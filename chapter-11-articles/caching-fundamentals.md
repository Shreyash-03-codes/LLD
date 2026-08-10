# Caching Fundamentals

## Learning Objectives

- Explain what a cache actually gives you, a second store for repeated reads, and what it does not: a fix for badly shaped queries.
- Name the three core dials, hit rate, TTL, and invalidation, and what breaking each one costs.
- Identify where a cache belongs on a read path and why caching the wrong layer is worse than not caching at all.

## Introduction

A cache is a fast second copy of some data, placed close to the reader so the slow source is hit less often. It trades memory and freshness for latency and throughput. The honest framing is that a cache buys you a stop: works only if the data is read far more than it is written, and only if you are willing to accept that the copy can be older than the source for a bounded window. Every design mistake you can make with a cache is a mistake about that trade.

## Problem Statement

A report endpoint takes 2 seconds because it sums millions of rows every call. The team's reflex is "add a cache." They wrap the whole response in a key-value store, which gives them a snappy endpoint and a wall of stale numbers. They now have a cache that is serving wrong data because the underlying rows changed and nothing invalidated it. The 2-second problem got replaced by a 2-ms problem that is quietly wrong. The failure is not that they cached. It is that they never decided how stale the answer could be, who updated it, and when it expired.

## Core Concept

A cache is a commitment between a consumer and a source, supervised by four dials. Every one of them is a decision, and the defaults are not safe.

- **Hit rate**: how often a read finds the value already resident. Zero hit rate and cache is dead weight that adds a hop.
- **TTL (time to live)**: how long a value can stay resident before it is considered stale. Long TTL raises freshness errors, short TTL raises misses.
- **Capacity / eviction**: what the cache forgets when full, LFU, LRU, FIFO. The policy decides which valuable keys survive pressure.
- **Invalidation**: how a source tells the cache that a value was updated and the copy is now stale. This is the dial that separates a cache from a redundant copy.

The reason these four show up in every discussion is that you cannot pick a single ideal for all of them. A cache is a compromise you describe precisely. The phrasing that carries the trade is "a bounded window of staleness," and it is that exact window that turns a good cache into a research project.

### Where it belongs on the path

The cache only pays off if it lives at the boundary that sees the repeated traffic. The wrong move is a blanket wrapper on the database connection. The pragmatic location sits between the service and the database, keyed on the thing the caller asks for, with the value being the assembled unit.

```java
public class InvoiceService {
    private final InvoiceRepository repository;
    private final Cache<String, Invoice> cache;   // TTL scoped

    public Invoice getInvoice(Long id) {
        String key = "invoice:" + id;
        try {
            return cache.resolveOrCompute(key, k -> repository.findById(id));
        }
        catch (CacheStoreMiss e) {
            return repository.findById(id);
        }
    }
}
```

Notice what is in the key and what is not. Cache keys on stable identity, not on the query string or the page title. The failure of absurdly keyed caches is that every reasonable variation becomes a miss. The value is the assembled unit, not the raw query result row.

### The stale window is the design

The central sentence you should be able to state for any cache: "for up to T seconds this reader sees a value that changed in the source." The choice is always over how stale things can get. A currency price is fine if 60 seconds old. An account balance is not fine if 2 seconds old. The value of caching is that you spend staleness to buy reads, and the business decides the price.

That property is what makes the cache not a fix for a slow query. If every read is genuinely unique, your hit rate lands near zero and the cache is a tax, not a win. Caching a slow query that nobody repeats is theater. You would be better off making the query fast.

### The four dials interacting

The dials do not move independently. A long TTL and a large capacity both push the hit rate up, but they also push the staleness window up, so a naive "make it bigger and longer" uses both dials to buy the same thing twice and pays twice. The design is not to maximize hit rate; it is to hit the freshness bound. If the product tolerates 5 minutes of staleness, you pick the TTL and the capacity not for maximum hits but to keep a miss cheap enough that the refresh under pressure does not thundering-herd back into the database. A cache that misses all at once under a traffic spike is a cache that converts a slow read into a database blackout.

The fix for that spike is a cache stampede guard. The least astonishing form is "single flight": when a key misses, only one caller computes the value and the rest wait on that single result, so they do not each splash the database. The guarded miss is the difference between one expensive recompute and a coordinated wallop of identical recomputes, and it is what keeps the endpoint alive through a cold start.

```java
private final LoadingCache<Long, Invoice> cache = Caffeine.newBuilder()
        .expireAfterWrite(Duration.ofMinutes(5))
        .build(id -> repository.findById(id));   // single flight on miss
```

Here `LoadingCache` already batches concurrent misses for the same key into one load, which is single-flight for free. When the value is gone from memory and many readers arrive at once, this is the knob that prevents the stampede.

### Local versus distributed

A local cache, like a Caffeine map inside one JVM, is free and instant, but every instance of your service has its own copy, and they go stale against each other. A distributed cache, like Redis, shares one copy across instances, which fixes cross-instance staleness but adds a network round trip to every hit. The trade is not "which is faster"; a Redis hit is still slower than a local hit. The real question is whether a stale value served from instance A while instance B just wrote the new one is acceptable. If it is, a local cache wins on cost. If it is not, you need the shared store. Most designs understate the local cache, which is odd because it is the cheapest latency win that exists.

```java
@Configuration
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cm = new CaffeineCacheManager();
        cm.setCaffeine(Caffeine.newBuilder().maximumSize(10_000)
                .expireAfterWrite(Duration.ofSeconds(30)));
        return cm;
    }
}
```

Written this way the whole bean quietly commits to the staleness window it names. The useful habit is that you can read a cache config and state both the duration and the capacity in one breath, because both are the design.

## Real Production Usage

Hibernate has a second-level cache for entities that rarely change, and Redis is the application-level store for computed results, rendered views, and high-read keys. Spring's `@Cacheable` binds the proxy to a `CacheManager` and lets a method's argument become the key, which is convenient and exactly how to forget that the annotation also has an eviction half (`@CacheEvict`) that you must use. Most production cache bugs are `@Cacheable` with the eviction left empty.

## Common Mistakes

1. **Caching the wrong thing and calling it a day.** Caching a query nobody repeats, or keying on a dynamic string, yields a hit rate near zero and a real tax.
2. **Ignoring staleness.** Picking a very long TTL means a copy goes stale and the product looks wrong for however long the value lives. TTL and business freshness belong in the same sentence.
3. **Only caching on read.** Writing values that are then read by a path that never evicts creates a hybrid broken copy.

## Interview Perspective

Interviewers ask "how do you design a cache?" and want the four dials, in order, plus the admission that the trade is controlled staleness, not myths about free acceleration. Weak: "cache everything with Redis." Strong: "key by stable identity, TTL by business freshness, evict on the write path, and only cache when the hit rate pays for the staleness risk." The tell of a strong answer is naming the stale window without being asked.

Follow-up: "If the write goes to the DB, what happens to the cache?" They want you to reach for the eviction/invalidate boundary, which is precisely the pattern whose swings the next article covers.

## Knowledge Check

1. An endpoint sums a 100-row join that nothing else reads. Why is caching likely a tax and what is the cheaper fix?
2. What are the two costs of a long TTL and what is the single phrase, "bounded window," trading against?
3. The cache stores whole Invoice but its key uses the user's timezone. What happens to your hit rate and why?

## Key Takeaways

- A cache is a trade of memory and bounded staleness for repeated-read speed; it is not a cure for a slow query.
- The four dials, hit rate, TTL, capacity, invalidation, are all decisions, and the defaults are not trustworthy.
- Where the cache sits and what the key is determine whether it reduces reads or just adds a hop.

## What's Next

All of these dials assume one decision was already made: where writes go relative to the cache. The next article drills into the three standard write policies, write-through, write-back, and cache-aside, and why the "just update the cache on write" instinct is the one that hurts most.

---

This article explains a cache as a second store built on four dials, hit rate, TTL, capacity, and invalidation, and the bounded staleness each trade costs. It claims that caching a rarely-repeated slow query is a tax, and that naming the stale window as a chosen trade is the position every design must take.