# Rate Limiting Algorithms

## Learning Objectives

- Compare token bucket, leaky bucket, fixed window, and sliding window on two axes: how they smooth bursts and how evenly they enforce the rate.
- Explain why distributed rate limiting changes the engineering: sharing a counter across instances instead of one counter living in a single process.
- Pick an algorithm by the shape of the traffic you must absorb, and know which algorithm is smooth and which permits bursts.

## Introduction

A rate limit is a contract: "you may make at most R requests per interval." It exists to shield a service from a burst that would overwhelm it, and clients from a service that fails from being overwhelmed. The word "algorithm" is in the title because the same contract can be enforced many ways, and the way you enforce it decides the shape of the traffic the client actually experiences. Some algorithms let a client burn N and burst through at full speed; others drain at a fixed rate no matter what. Choosing the shape is a real design decision, not a detail of the SDK.

## Problem Statement

A login endpoint starts offering tokens. A burst of legitimate users plus their automated retries arrives at the same second: 400 requests in one instant, then nothing for 50 seconds. The service can only process 40 a second. A team implements a fixed-window counter of 40 per minute. The first 40 pass, the next 360 are rejected, even though the rate averaged is trivially fine. The clients, told to retry, all retry at the next min tick, and the server is hammered in synchronized 40-request blocks. Meanwhile a malicious client discovers that a per-minute window resets at the minute mark, and schedules its 40 requests right after the reset, doubling its effective burst. The rate limit did exist and it shaped the traffic badly in both directions. The mistake was counting in fixed windows, where the shape of the stream was never the product of a design.

## Core Concept

All rate limiting is state, either per client or global, and a window of time over which the count is taken. The algorithms differ in what a client can do during a window, and what happens at a window boundary.

### Token bucket

The bucket holds up to `burst` tokens and fills at `rate` tokens per second. A request spends one token; if the bucket is empty the request waits or is dropped. The two dials are capacity (burst size) and fill rate (sustained rate). A client with a full bucket can fire a burst of up to `burst` at once, then settle into `rate` per second. This is the most popular shape because the two dials decouple "how hard may they spike" from "what may they sustain," and because it smooths drift gracefully; a short burst is soaked without a newly-window boundary cliff.

```java
public class TokenBucket {
    private final int capacity;
    private final double refillPerSecond;
    private double tokens;
    private long lastRefill;

    public TokenBucket(int capacity, double refillPerSecond) {
        this.capacity = capacity;
        this.refillPerSecond = refillPerSecond;
        this.tokens = capacity;
        this.lastRefill = System.nanoTime();
    }

    public synchronized boolean tryAcquire() {
        long now = System.nanoTime();
        double elapsedSec = (now - lastRefill) / 1_000_000_000.0;
        tokens = Math.min(capacity, tokens + elapsedSec * refillPerSecond);
        lastRefill = now;
        if (tokens >= 1.0) {
            tokens -= 1.0;
            return true;
        }
        return false;
    }
}
```

The refill is computed lazily on access, which is the trick that makes it cheap: no background thread, just math at the moment of the request. Tokens are credit that does not expire, which is the burst property.

### Leaky bucket

The leaky bucket is token bucket's controlled relative. Requests enter a queue of size `q`; a processor drains them at a fixed rate. The exit is an even stream, no bursts at all. The difference from token bucket matters: token bucket lets a client accumulate credit and spend it in a burst at the gateway; leaky bucket rejects overflow. Use the leaky where the shape must be smooth, a downstream that needs an even stream, a DB with a fixed insert budget, a batch that cannot absorb a spike. In Java, the pattern is a bounded queue plus a fixed-rate drain.

### Fixed window

The fixed window counts requests during a calendar window, minute boundary to boundary, and rejects beyond the cap. It is the simplest state: one counter, one expiry. Two flaws follow. First, the double-burst at the boundary: a client can do 40 requests at the end of minute 1 and 40 at the start of minute 2, a continuous stream of 80 in real time while never passing 40 "per window." Second, the whole client base can sync to the reset, creating a wall of retries exactly at the boundary. Fixed window is cheap and workable for small caps, but its burst behavior is the shape you usually don't want.

### Sliding window

The sliding window fixes the boundary problems: the window slides continuously, so 80 requests straddling the minute boundary are caught over any real rolling interval. Two implementations get different prices. The sliding window log stores a timestamp per request and counts the last `interval`; exact but O(requests) memory per client. The sliding window counter, cheaper, keeps two buckets (current and previous window, each with weight) and approximates the true count. The counter is the practical one: near-exact, constant memory.

| Algorithm | Burst behavior | Memory | Boundary behavior |
|-----------|----------------|--------|--------------------|
| Fixed window | entire cap resets per window | one counter | two consecutive bursts at the edge |
| Sliding log | exact window, sees through boundary | per-request list | exact |
| Sliding counter | mostly exact | 2 counters | good |
| Token bucket | burst to capacity, then rate | one float | smooth |
| Leaky bucket | no burst, constant drain | one queue | smooth |

Diagram: fixed window allows a doubled burst across the boundary; token bucket holds a burst, then a smooth rate.

<svg width="880" height="300" viewBox="0 0 880 300" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="rl" markerWidth="10" markerHeight="10" refX="8" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#333"/>
    </marker>
  </defs>

  <text x="30" y="40" font-family="Arial" font-size="13" fill="#222">Fixed window: the boundary splits one burst into two windows</text>
  <line x1="30" y1="120" x2="850" y2="120" stroke="#962828" stroke-width="1" stroke-dasharray="5,4"/>
  <rect x="30" y="60" width="400" height="40" fill="#eef2f7" stroke="#333"/>
  <rect x="430" y="60" width="420" height="40" fill="#eef2f7" stroke="#333"/>
  <rect x="30" y="60" width="40" height="40" fill="#2f6f3e"/>
  <text x="50" y="84" text-anchor="middle" font-family="Arial" font-size="12" fill="#fff">40</text>
  <rect x="300" y="60" width="40" height="40" fill="#2f6f3e"/>
  <text x="320" y="84" text-anchor="middle" font-family="Arial" font-size="12" fill="#fff">40</text>
  <rect x="430" y="60" width="40" height="40" fill="#2f6f3e"/>
  <text x="450" y="84" text-anchor="middle" font-family="Arial" font-size="12" fill="#fff">40</text>
  <rect x="700" y="60" width="40" height="40" fill="#2f6f3e"/>
  <text x="720" y="84" text-anchor="middle" font-family="Arial" font-size="12" fill="#fff">40</text>
  <rect x="185" y="60" width="80" height="40" fill="#2f6f3e"/>
  <text x="225" y="84" text-anchor="middle" font-family="Arial" font-size="12" fill="#fff">40</text>
  <rect x="615" y="60" width="80" height="40" fill="#2f6f3e"/>
  <text x="655" y="84" text-anchor="middle" font-family="Arial" font-size="12" fill="#fff">40</text>
  <text x="230" y="118" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">window 1</text>
  <text x="640" y="118" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">window 2</text>
  <text x="640" y="144" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">two bursts at the edge: 80 in real time</text>

  <text x="30" y="190" font-family="Arial" font-size="13" fill="#222">Token bucket: burst up to capacity, then refill at the rate</text>
  <rect x="30" y="215" width="80" height="55" rx="6" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="70" y="247" text-anchor="middle" font-family="Arial" font-size="11" fill="#8a6d00">bucket</text>
  <rect x="250" y="215" width="80" height="55" rx="6" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="290" y="247" text-anchor="middle" font-family="Arial" font-size="11" fill="#2f6f3e">take token</text>
  <rect x="470" y="215" width="80" height="55" rx="6" fill="#fdeeee" stroke="#962828" stroke-width="2"/>
  <text x="510" y="247" text-anchor="middle" font-family="Arial" font-size="11" fill="#962828">block</text>
  <path d="M 112 240 L 242 240" stroke="#333" stroke-width="2" fill="none" marker-end="url(#rl)"/>
  <path d="M 332 232 L 462 232" stroke="#2f6f3e" stroke-width="2" fill="none" marker-end="url(#rl)"/>
  <text x="396" y="222" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">token present</text>
  <path d="M 332 262 C 380 290 440 290 470 262" stroke="#962828" stroke-width="2" fill="none" marker-end="url(#rl)"/>
  <text x="408" y="286" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">empty</text>
  <path d="M 110 215 C 150 180 190 180 250 200" stroke="#8a6d00" stroke-width="2" fill="none" marker-end="url(#rl)"/>
  <text x="185" y="178" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">refill rate</text>
</svg>

The two rows say it in one look. The fixed window lets a client spend the tail of one window and the head of the next at full speed, so the real rate doubles across the boundary. The token bucket spends the burst it had saved and then settles into the refill rate, so the shape of the stream is the thing you actually designed.

### Where the state lives

A rate limiter that runs inside one process keeps the counter in memory, cheap and exact, but a multi-instance deploy has one limiter per instance, so each instance allows its own R and the service allows N×R. The fix is a shared store: Redis with an `INCRBY` and a TTL gives a single global cap at the cost of a network hop on every limited request, plus its own race, two instances both decrementing a near-empty bucket can both pass. The workable shape in production is a per-instance token bucket for the everyday cap, cheap and fast, with a shared counter in Redis for the security-critical hard ceiling, where exactness costs more but matters more. The two limits are different things and should not share a knob.

You also decide where the line is drawn per client. A per-client cap at 100 r/s is precise and needs a store keyed by client; a global cap protects the whole service at the risk of blocking one fast consumer; per-IP is the coarsest gate, easy to spoof. Which clients get which capacity is a product and security decision, and getting it wrong is either a fraud hole or a huge metering bill.

## Real Production Usage

The gateway is the usual home: NGINX has `limit_req` with a token-bucket-style `burst`/`nodelay`, and API gateways like Kong and Spring Cloud Gateway (via its `RequestRateLimiter` filter backed by Redis) implement a fixed or sliding window. Inside the service, Resilience4j's `RateLimiter` is the in-process option. You end up with the layered setup of the previous section: in-process for the resilience budget, Redis for the global cap. The engineering decision is where your guarantee lives, and the honest answer in Java is usually "both, for different reasons."

## Common Mistakes

1. **Rate limiting per instance and believing it is global.** In a fleet of 20, each instance at 100/min is an effective 2000/min ceiling, and the only place aware of that is the downstream that is dying. The limit that matters is what the fleet actually sends, not what one config says.
2. **Fixed windows by default.** The minute boundary is the reason a retry burst can synchronize on the reset and survive as double bursts. Use token bucket where bursts are legitimate and sliding where the interval must be exact.
3. **Ignoring the retry shape of the rejection.** When you reject calls, clients will retry. If every client retries on the same aligned cue, you built a synchronized hammer. A 429 with a jittered Retry-After spreads the herd instead of aligning it.

## Interview Perspective

The prompt "design a rate limiter" is a shape question: capacity, burst, window, and where the state lives. Strong: "a token bucket per client for a burst-plus-rate shape, and a shared Redis counter for the global hard cap, with a 429 plus a jittered retry-after as the rejection path." Weak: "I set a counter and cap it at 100." The probes that separate candidates are "what does a token bucket actually smooth" and "how is a sliding window different from a fixed one," either of which rules out someone who only memorized the name.

## Knowledge Check

1. A client spends the last second of minute 1 and the first of minute 2 at full throttle. Which algorithms let the double through, and how does a sliding window catch the effect of the count?
2. A fleet of 6 instances each runs a token limiter at 100/min. An upstream mandates an absolute ceiling of 300/min. Where has the enforcement moved, and where does the fix belong?
3. A rejected client is told "retry in 5 seconds" and every client obeys. Explain why the response is the problem, and what response shape spreads the retries.

## Key Takeaways

- Token bucket lets bursts in, leaky bucket never does, fixed window doubles at the boundary, sliding window approximates the exact with cheap state.
- The state is the design: in-process is fast but not global, a shared store is global but costs a hop and adds a race.
- The 429 is part of the algorithm; what you tell the client to do decides whether a rejection is a spread or a synchronized spike.

## What's Next

Rate limiting is a policy fought at the boundary of demand, and it's configured globally, in the time of collection, in prod. The next article is the feature flip switch that lets you act on the demand at much finer granularity than an RPS cap: feature flags, how you gate a small percentage of traffic to a new behavior without a new deploy, and how the configuration lifecycle and the kill switch are themselves subjects of robustness.

---

This article explains rate limiting as a design of burst shape and window, comparing token bucket, fixed, and sliding window. It argues the algorithm decides whether the boundary is a burst enabler, and the storage decides whether the limit is global or per instance.
