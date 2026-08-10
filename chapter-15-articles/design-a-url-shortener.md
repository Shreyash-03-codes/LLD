# Design a URL Shortener

## Learning Objectives

- Learn that the entire system is a single mapping from short key to long URL, and that every interesting decision is about how you generate and store the keys.
- Compare key generation strategies, sequential encoding versus random, and know what each one costs in collisions, guessability, and hot-spot contention.
- Model the read path and write path separately, because a shortener is extreme read-heavy, and the storage design follows from that.

## Introduction

The URL shortener is the first case study in this chapter with almost no domain rules. No overlap checks, no states, no turn loops. A shortener does two things: it takes a long URL and hands back a short key, and it takes a short key and hands back the long URL. That is the whole system, and the interview is not about the features, it is about the mapping. Interviewers ask this because the simplicity exposes pure engineering judgment: how you generate keys, how you avoid collisions, how you make the read path fast, and how you answer the "what if we run out of keys" question. It is also a small enough system that a bad decision, like a global counter with no thought for contention, is visible in one sentence.

## Requirements Gathering

Functional requirements:

- A client submits a long URL and receives a short key.
- A client resolves a short key and is redirected to the long URL.
- The same long URL may be shortened more than once, and each shortener call yields its own key.
- Short keys can expire after a configurable lifetime.
- Invalid or expired keys return a not-found result.

Non-functional requirements:

- The read path, key to URL, must be fast, because it runs on every click and dominates traffic.
- Key generation must never produce a duplicate for an unexpired key, under any load.

Assumptions to state out loud: no custom aliases or vanity URLs, no analytics or click tracking, no user accounts, single-region storage with no cross-datacenter replication story, and expiry is best-effort rather than exact. Cut analytics and cut vanity URLs. The interviewer wants the mapping, and vanity URLs are a whole second mapping that changes key generation from pure to user-chosen, which is a different problem.

## Identifying Core Entities

The entity list is small, and one of them is a decision point disguised as a class.

| Entity | One-line responsibility |
| --- | --- |
| `ShortKeyGenerator` | Produces the next short key under a chosen strategy. |
| `UrlStore` | The key-to-URL mapping, with create, resolve, and delete operations. |
| `ShortenService` | The facade exposing shorten and resolve. |
| `ShortUrl` | The record: key, long URL, creation time, expiry. |

The decision point is `ShortKeyGenerator`. Whether it is a counter encoder, a random generator, or something else is the design question of the whole case study, and it should be an interface with at least two honest implementations so the trade-off discussion has somewhere to live.

## Class Design

Start with the key. The classic encoding is base62, which gives you 62^7 keys for a 7-character string, roughly 3.5 trillion. The encoding is arithmetic: divide by 62 repeatedly, map remainders to the alphabet. It is the same idea as converting a number to a new base, and it is the one piece of arithmetic in the whole case study.

```java
public class Base62 {
    private static final String ALPHABET = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

    public static String encode(long value) {
        StringBuilder sb = new StringBuilder();
        while (value > 0) {
            sb.append(ALPHABET.charAt((int) (value % 62)));
            value /= 62;
        }
        return sb.reverse().toString();
    }

    public static long decode(String key) {
        long value = 0;
        for (char c : key.toCharArray()) {
            value = value * 62 + ALPHABET.indexOf(c);
        }
        return value;
    }
}
```

Now the two key strategies. The sequential strategy encodes a counter: every shorten call takes the next number, encodes it, and that is the key. It is collision-free by construction and the keys are the shortest possible for their age, but the counter is a hot spot, and the keys are sequential, which means they are guessable and enumerable.

```java
public class CounterKeyGenerator implements ShortKeyGenerator {
    private final AtomicLong counter = new AtomicLong(0);

    public String nextKey() {
        return Base62.encode(counter.incrementAndGet());
    }
}
```

The random strategy generates a random value and re-rolls on collision. The keys are unguessable and there is no hot counter, but every insert must check for collision, and at very high volume the collision rate rises as the key space fills.

```java
public class RandomKeyGenerator implements ShortKeyGenerator {
    private final SecureRandom random = new SecureRandom();
    private final UrlStore store;

    public RandomKeyGenerator(UrlStore store) { this.store = store; }

    public String nextKey() {
        while (true) {
            String key = Base62.encode(random.nextLong(62_000_000_000L));
            if (!store.exists(key)) {
                return key;
            }
        }
    }
}
```

The interesting interview move is to name the third strategy before being asked: the counter strategy can be made horizontally scalable with a coordinator that hands each server a range of numbers, or by offsetting each server's counter, so "what if we run out" and "what if one server is the counter" both have answers.

`UrlStore` is the mapping. The read path is a `Map` lookup in the interview version; in production it is a key-value store or a database row indexed by key, with a cache in front. The design keeps that split explicit by making `resolve` the read path and `create` the write path, because the two have different performance profiles.

```java
public class UrlStore {
    private final ConcurrentHashMap<String, ShortUrl> store = new ConcurrentHashMap<>();

    public Optional<ShortUrl> create(ShortUrl url) {
        return store.putIfAbsent(url.getKey(), url) == null
                ? Optional.of(url)
                : Optional.empty();
    }

    public Optional<ShortUrl> resolve(String key) {
        ShortUrl url = store.get(key);
        if (url != null && url.isExpired()) {
            store.remove(key);
            return Optional.empty();
        }
        return Optional.ofNullable(url);
    }

    public boolean exists(String key) { return store.containsKey(key); }
}
```

`putIfAbsent` is the collision guard on the write path: even a generator that thinks it produced a fresh key cannot overwrite an existing one, which is the safety net under every strategy. `ShortUrl` carries the expiry and answers `isExpired`, so expiry is a field and a check, not a sweeper thread.

```java
public class ShortUrl {
    private final String key;
    private final String longUrl;
    private final Instant createdAt;
    private final Instant expiresAt;

    public ShortUrl(String key, String longUrl, Instant expiresAt) {
        this.key = key;
        this.longUrl = longUrl;
        this.createdAt = Instant.now();
        this.expiresAt = expiresAt;
    }

    public boolean isExpired() {
        return expiresAt != null && Instant.now().isAfter(expiresAt);
    }

    public String getKey() { return key; }
    public String getLongUrl() { return longUrl; }
}
```

`ShortenService` ties it together: generate a key, build a record, store it, and return the short URL. The resolve path is one lookup and a redirect decision.

```java
public class ShortenService {
    private final ShortKeyGenerator generator;
    private final UrlStore store;

    public ShortenService(ShortKeyGenerator generator, UrlStore store) {
        this.generator = generator;
        this.store = store;
    }

    public Optional<String> shorten(String longUrl) {
        String key = generator.nextKey();
        ShortUrl url = new ShortUrl(key, longUrl, Instant.now().plus(Duration.ofDays(365)));
        return store.create(url).map(u -> "https://short.example/" + u.getKey());
    }

    public Optional<String> resolve(String key) {
        return store.resolve(key).map(ShortUrl::getLongUrl);
    }
}
```

Diagram: one mapping, two decisions. Top, the key generation strategies and what each costs. Bottom, the write path and the read path, where reads are the business.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 920 440" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="aha" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
  </defs>
  <rect width="920" height="440" fill="#ffffff"/>

  <text x="460" y="26" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">One mapping, two decisions</text>

  <text x="30" y="76" font-size="14" font-weight="bold" fill="#1f2937">Key generation strategies</text>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12.5" fill="#1f2937">
    <rect x="20" y="92" width="280" height="110" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
    <text x="160" y="116" text-anchor="middle" font-weight="bold" fill="#1e3a8a">Sequential (counter)</text>
    <text x="160" y="138" text-anchor="middle">1, 2, 3 → '1', '2', '3'</text>
    <text x="160" y="162" text-anchor="middle" fill="#15803d">✓ collision-free · short keys</text>
    <text x="160" y="184" text-anchor="middle" fill="#b91c1c">✗ guessable · hot counter</text>

    <rect x="320" y="92" width="280" height="110" rx="8" fill="#e0e7ff" stroke="#6366f1"/>
    <text x="460" y="116" text-anchor="middle" font-weight="bold" fill="#3730a3">Range allocation</text>
    <text x="460" y="138" text-anchor="middle">coordinator hands each server a range</text>
    <text x="460" y="162" text-anchor="middle" fill="#15803d">✓ scales the counter across servers</text>
    <text x="460" y="184" text-anchor="middle" fill="#4338ca">the answer to "what if we run out of keys"</text>

    <rect x="620" y="92" width="280" height="110" rx="8" fill="#fef3c7" stroke="#f59e0b"/>
    <text x="760" y="116" text-anchor="middle" font-weight="bold" fill="#92400e">Random</text>
    <text x="760" y="138" text-anchor="middle">random value → re-roll on collision</text>
    <text x="760" y="162" text-anchor="middle" fill="#15803d">✓ unguessable · no hot counter</text>
    <text x="760" y="184" text-anchor="middle" fill="#b91c1c">✗ collision check per insert</text>
  </g>

  <text x="460" y="238" text-anchor="middle" font-size="14" font-weight="bold" fill="#1f2937">Read path vs write path</text>
  <text x="30" y="264" font-size="12.5" font-weight="bold" fill="#1e3a8a">WRITE</text>
  <text x="30" y="354" font-size="12.5" font-weight="bold" fill="#7c3aed">READ</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#aha)">
    <line x1="140" y1="296" x2="176" y2="296"/>
    <line x1="340" y1="296" x2="376" y2="296"/>
    <line x1="560" y1="296" x2="596" y2="296"/>
    <line x1="140" y1="386" x2="176" y2="386"/>
    <line x1="340" y1="386" x2="376" y2="386"/>
    <line x1="560" y1="386" x2="596" y2="386"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12.5" fill="#1f2937">
    <rect x="20" y="274" width="120" height="44" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="80" y="300" text-anchor="middle" font-weight="bold" fill="#334155">Client</text>
    <rect x="180" y="274" width="160" height="44" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="260" y="295" text-anchor="middle" font-weight="bold" fill="#334155">ShortenService.shorten</text>
    <text x="260" y="310" text-anchor="middle" font-size="11" fill="#64748b">generate key</text>
    <rect x="380" y="274" width="180" height="44" rx="8" fill="#fffbeb" stroke="#f59e0b"/>
    <text x="470" y="295" text-anchor="middle" font-weight="bold" fill="#92400e">UrlStore.create</text>
    <text x="470" y="310" text-anchor="middle" font-size="11" fill="#b45309">putIfAbsent — collision net</text>
    <rect x="600" y="274" width="200" height="44" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="700" y="300" text-anchor="middle" font-weight="bold" fill="#14532d">short.example/abc123</text>

    <rect x="20" y="364" width="120" height="44" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="80" y="390" text-anchor="middle" font-weight="bold" fill="#334155">Client</text>
    <rect x="180" y="364" width="160" height="44" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="260" y="385" text-anchor="middle" font-weight="bold" fill="#334155">ShortenService.resolve</text>
    <text x="260" y="400" text-anchor="middle" font-size="11" fill="#64748b">lookup by key</text>
    <rect x="380" y="364" width="180" height="44" rx="8" fill="#eef2ff" stroke="#6366f1"/>
    <text x="470" y="385" text-anchor="middle" font-weight="bold" fill="#3730a3">UrlStore.resolve</text>
    <text x="470" y="400" text-anchor="middle" font-size="11" fill="#4338ca">lazy expiry check + cache</text>
    <rect x="600" y="364" width="200" height="44" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="700" y="390" text-anchor="middle" font-weight="bold" fill="#14532d">301 redirect → long URL</text>
  </g>

</svg>
```

That is the whole system. The interview is not in the lines, it is in the discussion of why the generator is an interface and what each strategy costs.

## Design Patterns Used

The pattern here is the Strategy pattern on key generation, and it is not decorative: the two strategies genuinely differ in behavior and correctness properties, and making the choice a strategy is what lets you discuss them side by side and swap them without touching the service. That is the textbook case of Strategy earning its keep. Beyond that, resist everything. There is no Factory for URLs, no Builder for the record (a constructor is fine), no Observer for analytics (cut from scope). The one structural idea worth naming that is not a GoF pattern is the collision safety net: `putIfAbsent` on the write path means the generator can be sloppy about collision checking and the store still cannot overwrite. That separation of "generate" from "guarantee unique" is the kind of layering interviewers notice.

## Handling Edge Cases / Concurrency

The write path has the one real concurrency story: two servers shorten simultaneously with the counter strategy. If the counter is a single process, `AtomicLong` handles it. If it is distributed, the shared counter is a hot spot and a coordination point, which is why the range-allocation strategy exists. The random strategy moves the concurrency problem to the store: two servers could generate the same random key, which is why `create` must be a compare-and-set, not a put. The `putIfAbsent` is the answer to "what happens when two shorten calls collide," and the walkthrough version is "the second call gets an empty result from `create`, and the caller retries with a new key," which is exactly what `RandomKeyGenerator.nextKey` does with its loop.

The expiry edge is worth naming: an expired key is removed lazily on resolve rather than by a sweeper, so the store can contain expired entries briefly, and the `resolve` path must check expiry on every read. The alternative, a periodic sweep, adds a background component for a problem the lazy check already solves.

## Common Mistakes

The most common mistake is treating key generation as a throwaway line. The candidate writes `UUID.randomUUID().toString().substring(0, 7)` and moves on, having chosen a strategy, its collision profile, its guessability, and its length, without saying one word about any of them. The generator is the case study. Skip it and the interview is over in ten minutes.

The second mistake is a global counter without a story. `static long counter` with `counter++` answers the "concurrent shortens" question with a race, and the "what if two servers" question with silence. Either an `AtomicLong`, which is the single-process answer, or a range-allocation scheme, which is the distributed answer, must be stated.

The third mistake is ignoring the read-heavy shape. A design where every resolve recomputes the key, or scans for the URL by value, has missed that the shortener's traffic is clicks, not creates, and the read path is a lookup by key with a cache in front. The write path is clever; the read path must be boring.

## Interview Perspective

A weak answer is a single `UrlShortener` class with a `HashMap` and `uuid.substring`, and no discussion. The interviewer asks "what if two servers shorten at the same time" and gets a shrug, then "what if we run out of keys" and gets silence. The system has no decisions in it because none were made.

A strong answer says "the design is one mapping, and the interesting choice is key generation: sequential is short and collision-free but guessable and a hot counter, random is unguessable but needs a collision net, and the store's compare-and-set is what makes either strategy safe." That paragraph answers the four standard follow-ups at once. The predictable twists: "what if keys must not be guessable" (switch the generator, the store does not change), "what if we need to scale the counter" (range allocation, or per-server offsets), "what if a URL is shortened a million times" (a million keys, which is the requirement as stated, and a cache-friendly count model if we want dedup, which was cut at the start). The strongest candidates mention the cache in front of the store unprompted, because a shortener's read path is the whole business.

## Knowledge Check

1. Two servers shorten URLs concurrently under the counter strategy. Describe the two ways to make the counter safe, and what breaks if neither is used.
2. The random strategy generates a key that already exists in the store. Trace the exact call sequence that handles this, from the generator to the store and back.
3. A resolve request arrives for a key that expired an hour ago. Walk through `UrlStore.resolve` and state what the caller observes, and why a lazy expiry check is sufficient for this system.

## Key Takeaways

- The system is one mapping from key to URL. Every decision is about that mapping.
- Key generation is a Strategy: sequential encodes a counter, random re-rolls on collision, and each has real costs to state out loud.
- The store's compare-and-set is the collision safety net; the generator is allowed to be optimistic only because the store is strict.
- Reads are the business. A fast key lookup with a cache in front is the whole read path.
- Expiry is a field and a lazy check, not a sweeper thread.

## What's Next

The URL shortener was a pure mapping, and its decisions were all about keys and reads. Splitwise is a return to real domain rules, and the design is about money distribution: who owes whom, and how a debt graph collapses into the fewest transactions.

---

This article explains how to design a URL shortener around the key-to-URL mapping and the key generation strategy every decision hangs off. Its point of view is that the read path is the business, and a boring cache-friendly lookup beats cleverness in the key generator.
