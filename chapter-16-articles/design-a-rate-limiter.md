# Design a Rate Limiter

## Learning Objectives

- Learn the four canonical algorithms, fixed window, sliding window, token bucket, and sliding window counter, and what each one costs in memory, accuracy, and burst behavior.
- Decide where the limiter lives, client, gateway, or server, and know why the placement is a trust and performance decision, not a style choice.
- Model the distributed version, and see why a single-node limiter is a different design from one that must agree across replicas.

## Introduction

The rate limiter is the case study where the domain is one sentence, "a client may make at most N requests per window," and the design is everything. There is no state machine, no inventory, no pipeline. The entire system is a counter and a clock, and the interview is about which algorithm you use to count, where you put the count, and what happens when the count is wrong. Interviewers ask this because it is the rare LLD topic that is genuinely algorithmic, and because every candidate claims to know it while most can name only "sliding window" and cannot say what it costs. The case study rewards precision: the algorithm, the trade-off, and the failure mode, in that order, with no hedging.

## Requirements Gathering

Functional requirements:

- A client identified by a key (user, IP, API token) can make at most N requests per time window.
- Requests over the limit are rejected with a clear signal, usually HTTP 429, and the remaining quota can be communicated.
- The limit can be configured per key and per endpoint.

Non-functional requirements:

- The check must add negligible latency to the request path, single-digit milliseconds.
- The limiter must be correct across multiple server instances if deployed behind a load balancer.

Assumptions to state out loud: limits are per minute at the simplest tier and per second at the strictest, no prioritization of certain clients over others, no cost weighting per endpoint complexity, and the limiter rejects rather than queues. Cut prioritization and cut adaptive limits. The interviewer wants the counting algorithms, and priority queues are a scheduler's problem, not a limiter's.

## Identifying Core Entities

The entity list is short, and one entity is a family of implementations rather than a single class.

| Entity | One-line responsibility |
| --- | --- |
| `RateLimiter` | The interface: check a key and decide allow or deny. |
| `WindowCounter` | The per-key count data structure an algorithm uses. |
| `LimiterConfig` | The rule: max requests and window duration per key scope. |
| `RateLimitRuleStore` | Holds the configured limits and looks them up by key. |
| `LimiterMiddleware` | The request-path hook that applies the check. |

The interesting design decision is that `RateLimiter` should be an interface with several implementations, because the four algorithms are genuinely different and the choice is the case study.

## Class Design

Start with the config and the interface, then the algorithms.

```java
public class LimiterConfig {
    private final int maxRequests;
    private final long windowSeconds;

    public LimiterConfig(int maxRequests, long windowSeconds) {
        this.maxRequests = maxRequests;
        this.windowSeconds = windowSeconds;
    }

    public int getMaxRequests() { return maxRequests; }
    public long getWindowSeconds() { return windowSeconds; }
}

public interface RateLimiter {
    boolean allow(String key);
}
```

The fixed window algorithm is the simplest: count requests per key per window, reset the count when the window rolls over. One counter per key, one timestamp. The problem is the window boundary: a client can do N requests in the last second of one window and N more in the first second of the next, doubling the effective rate for two consecutive seconds.

```java
public class FixedWindowLimiter implements RateLimiter {
    private final LimiterConfig config;
    private final Map<String, Long> counts = new ConcurrentHashMap<>();
    private final Map<String, Long> windowStarts = new ConcurrentHashMap<>();

    public FixedWindowLimiter(LimiterConfig config) { this.config = config; }

    public boolean allow(String key) {
        long now = System.currentTimeMillis();
        long window = now / (config.getWindowSeconds() * 1000);
        Long start = windowStarts.putIfAbsent(key, window);
        if (start == null) {
            counts.put(key, 1L);
            return true;
        }
        if (start != window) {
            windowStarts.put(key, window);
            counts.put(key, 1L);
            return true;
        }
        return counts.merge(key, 1L, Long::sum) <= config.getMaxRequests();
    }
}
```

Diagram: the fixed window's boundary spike. Each window alone is under the limit, but requests clustered around the rollover double the effective rate.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 420" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="ah8" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#dc2626"/>
    </marker>
  </defs>
  <rect width="900" height="420" fill="#ffffff"/>

  <text x="450" y="28" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">The fixed window's boundary spike</text>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="13" fill="#1f2937">
    <rect x="20" y="118" width="420" height="64" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
    <text x="32" y="140" font-weight="bold" fill="#1e3a8a">Window 1 · 0:00–1:00</text>
    <text x="32" y="170" font-size="12.5" fill="#14532d">100 requests — within limit ✓</text>
    <rect x="460" y="118" width="420" height="64" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
    <text x="660" y="140" font-weight="bold" fill="#1e3a8a">Window 2 · 1:00–2:00</text>
    <text x="660" y="170" font-size="12.5" fill="#14532d">100 requests — within limit ✓</text>
  </g>

  <g fill="#1e3a8a">
    <rect x="330" y="158" width="9" height="8"/>
    <rect x="340" y="156" width="9" height="10"/>
    <rect x="350" y="154" width="9" height="12"/>
    <rect x="360" y="152" width="9" height="14"/>
    <rect x="370" y="150" width="9" height="16"/>
    <rect x="380" y="148" width="9" height="18"/>
    <rect x="390" y="146" width="9" height="20"/>
    <rect x="400" y="144" width="9" height="22"/>
    <rect x="410" y="142" width="9" height="24"/>
    <rect x="420" y="140" width="9" height="26"/>
    <rect x="480" y="140" width="9" height="26"/>
    <rect x="490" y="142" width="9" height="24"/>
    <rect x="500" y="144" width="9" height="22"/>
    <rect x="510" y="146" width="9" height="20"/>
    <rect x="520" y="148" width="9" height="18"/>
    <rect x="530" y="150" width="9" height="16"/>
    <rect x="540" y="152" width="9" height="14"/>
    <rect x="550" y="154" width="9" height="12"/>
    <rect x="560" y="156" width="9" height="10"/>
    <rect x="570" y="158" width="9" height="8"/>
  </g>

  <line x1="440" y1="108" x2="440" y2="200" stroke="#dc2626" stroke-width="2" stroke-dasharray="6 4"/>
  <text x="440" y="224" text-anchor="middle" font-size="12.5" fill="#dc2626" font-weight="bold">boundary — count resets</text>

  <line x1="350" y1="250" x2="545" y2="250" stroke="#dc2626" stroke-width="2" marker-start="url(#ah8)" marker-end="url(#ah8)"/>
  <text x="448" y="242" text-anchor="middle" font-size="13.5" fill="#b91c1c" font-weight="bold">200 requests in ≈2 seconds — the spike</text>

  <text x="450" y="296" text-anchor="middle" font-size="15" font-weight="bold" fill="#1f2937">Why the other algorithms are immune</text>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12.5" fill="#1f2937">
    <rect x="20" y="312" width="280" height="72" rx="8" fill="#f0fdf4" stroke="#bbf7d0"/>
    <text x="160" y="336" text-anchor="middle" font-weight="bold" fill="#14532d">Token bucket</text>
    <text x="160" y="356" text-anchor="middle">burst ≤ bucket capacity,</text>
    <text x="160" y="372" text-anchor="middle">then steady refill ✓</text>
    <rect x="310" y="312" width="280" height="72" rx="8" fill="#f0fdf4" stroke="#bbf7d0"/>
    <text x="450" y="336" text-anchor="middle" font-weight="bold" fill="#14532d">Sliding window log</text>
    <text x="450" y="356" text-anchor="middle">counts in the true rolling</text>
    <text x="450" y="372" text-anchor="middle">window — exact, memory-hungry ✓</text>
    <rect x="600" y="312" width="280" height="72" rx="8" fill="#f0fdf4" stroke="#bbf7d0"/>
    <text x="740" y="336" text-anchor="middle" font-weight="bold" fill="#14532d">Sliding window counter</text>
    <text x="740" y="356" text-anchor="middle">weights previous window by</text>
    <text x="740" y="372" text-anchor="middle">elapsed fraction — close, 2 counters ✓</text>
  </g>

</svg>
```

The token bucket algorithm fixes the boundary problem with a different model. Instead of counting per window, a bucket holds tokens, tokens refill at a steady rate, and each request takes a token. The burst behavior is what makes it attractive: a full bucket allows an immediate burst up to its capacity, then sustains the refill rate. One counter per key and one timestamp still, same memory as fixed window.

```java
public class TokenBucketLimiter implements RateLimiter {
    private final LimiterConfig config;
    private final Map<String, Long> tokens = new ConcurrentHashMap<>();
    private final Map<String, Long> lastRefill = new ConcurrentHashMap<>();

    public TokenBucketLimiter(LimiterConfig config) { this.config = config; }

    public boolean allow(String key) {
        long now = System.currentTimeMillis();
        synchronized (this) {
            long last = lastRefill.getOrDefault(key, now);
            long elapsedMs = now - last;
            long refill = elapsedMs * config.getMaxRequests() / (config.getWindowSeconds() * 1000);
            long current = Math.min(config.getMaxRequests(),
                    tokens.getOrDefault(key, 0L) + refill);
            tokens.put(key, current);
            lastRefill.put(key, now);
            if (current <= 0) {
                return false;
            }
            tokens.put(key, current - 1);
            return true;
        }
    }
}
```

The sliding window log algorithm is the accurate but memory-hungry one: keep a timestamp per request per key, and count how many fall inside the window. Exact, no boundary spike, but a busy key accumulates a timestamp list, and the memory and the cleanup cost grow with the request rate.

The sliding window counter is the compromise that production systems actually use: keep the count for the current window and the previous window, and estimate the current rate by weighting the previous window's count by the elapsed fraction. It is not exact, but it is close, and it costs two counters per key.

The candidate who can lay out all four, with the memory and accuracy trade-off of each, has finished the case study. The candidate who knows only one has just begun it.

`LimiterMiddleware` is where the limiter meets the request path. The placement question, client, gateway, or server, is a real design decision. Client-side limiting trusts the client, which only works for honest clients. Server-side limiting is per-instance, which breaks behind a load balancer unless the count is shared. Gateway or distributed limiting is where the shared count lives in Redis, and the check becomes a `GET` and an atomic `INCR` with an expiry.

```java
public class LimiterMiddleware {
    private final RateLimiter limiter;
    private final Map<String, LimiterConfig> rules;

    public boolean checkRequest(String key, String endpoint) {
        LimiterConfig config = rules.get(endpoint);
        if (config == null) {
            return true;
        }
        return limiter.allow(key);
    }
}
```

## Design Patterns Used

The pattern here is the Strategy pattern applied to algorithms, and it is the textbook case: four interchangeable counting strategies with identical signatures and different behavior, selected by configuration. The interface exists precisely so the algorithm choice is a wiring decision, not a rewrite. That is Strategy as it is supposed to be used. Beyond that, resist. There is no Observer for threshold alerts (not in scope), no Facade needed beyond the middleware, and no Decorator for layering limiters, because the case study is one limiter, not a composition exercise.

## Handling Edge Cases / Concurrency

The concurrency story is the per-key check-and-increment. On a single node, the `synchronized` in the token bucket, or the atomic `merge` in the fixed window, closes the race. Across nodes, the count must be shared, and the production answer is Redis with an atomic increment and an expiry, or a Lua script that does the read-and-increment in one shot. The candidate who says "Redis INCR with EXPIRE" and can explain why the increment must be atomic, because two replicas reading the same count and both deciding to allow is the oversell, has the distributed story.

The boundary edges are algorithmic: the fixed window's burst at the boundary, the token bucket's allowed burst, and the sliding window counter's approximation error. Each is a stated cost of its algorithm, and the walkthrough should name the boundary behavior of whichever algorithm was chosen. The memory edge: the sliding window log's per-request timestamps grow without bound for hot keys, which is exactly why the two-counter version exists.

## Common Mistakes

The most common mistake is naming "sliding window" and stopping. The interviewer asks "how does it work" and the candidate describes something that is either a sliding window log, which has the memory problem, or a sliding window counter, which is approximate, and cannot say which. The four algorithms are the content of this case study. Not knowing which one you drew is a failure at the first question.

The second mistake is the single-node limiter presented as production-ready. The candidate draws an in-memory `HashMap` and does not mention that behind a load balancer, three replicas would each allow the full quota. The distributed count is not an extension, it is the difference between a limiter that works and a limiter that merely exists.

The third mistake is conflating the limiter with the queue. A rate limiter rejects, it does not buffer. The candidate who adds a waiting queue has designed a throttler, which is a different product with a different failure mode, and the interviewer's "what happens when the queue is full" is a question the limiter should never have to answer.

## Interview Perspective

A weak answer is a `HashMap<String, Integer>` counting calls in the last hour with no window concept, and the interviewer's "reset the count when?" produces a shrug. Or a confident "sliding window" with no mechanism. The case study is over before the trade-offs were even offered.

A strong answer says "the limiter is a strategy, and the choice is between four algorithms with real costs: fixed window is cheap with a boundary spike, token bucket allows controlled bursts, sliding window log is exact but memory-hungry, and the sliding window counter is the production compromise." Follow-ups to expect: "which one would you pick" (token bucket for API backends, because bursts are real and the refill is steady, stated with the memory cost of one counter per key), "how do you make it distributed" (Redis atomic increment with expiry, and the two-replica oversell scenario that motivates it), "what happens when Redis is down" (fail open or fail closed, a stated policy choice, and the strong candidate takes one and says what it costs). The strongest candidates volunteer the boundary spike of the fixed window unprompted.

## Knowledge Check

1. A client sends 100 requests at second 59 of a minute and 100 more at second 1 of the next minute, under a 100-per-minute fixed window. Explain what happened and which algorithm class is immune to it.
2. A token bucket with capacity 10 and a refill of 5 per second is empty at 12:00:00. A client sends 20 requests immediately, then 4 requests every second. Walk through which requests succeed and which are rejected.
3. A limiter runs on three replicas behind a load balancer with a per-instance count. Describe the concrete way a client defeats it, and what the Redis version changes about the count.

## Key Takeaways

- Four algorithms, four cost profiles. Know them all and which one you are actually describing.
- The token bucket is the production default for API backends: bounded bursts, steady refill, one counter per key.
- A limiter rejects, it does not queue. A queue is a different product.
- Distributed means shared count with atomic increments, or the load balancer hands out a quota per replica.
- The Redis-down policy is a declared choice: fail open costs a little abuse, fail closed costs availability. Take one.

## What's Next

The rate limiter made the counting algorithm the whole case study. The job scheduler keeps the time dimension but reverses the flow: instead of rejecting excess work, it takes work and decides when it runs, and the heap, not the counter, is the data structure at the center.

---

This article explains how to design a rate limiter by comparing the four counting algorithms and their memory and burst trade-offs. Its point of view is that naming sliding window and stopping is a failure, and the token bucket is the production answer.
