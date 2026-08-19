# Write-Through vs Write-Back vs Cache-Aside

## Learning Objectives

- Distinguish the three write policies by what they do on a write and where the risk of inconsistency lives.
- Trace the read and write path of each policy and name its typical failure mode.
- Choose a policy for a workload and justify it in terms of consistency cost and throughput.

## Introduction

The previous article gave you the cache dials. This one is about the moment of writing: where does the new value go, first, second, and third. The three standard policies, write-through, write-back, and cache-aside, differ on which store owns the truth at any instant and how long a divergence between cache and database is allowed to survive. The policy is the contract for that window.

## Problem Statement

Your service updates a user's profile. You write to the database and, helpfully, also to the cache, in that order. The write fails after the database commit, or the order is reversed and the cache ends up holding a stale value. Either way, the next reader gets data that contradicts the database, and nobody knows which store to trust. The wrong policy makes the cache a source of quietly wrong reads. The fix is a policy that decides the truth order and the failure behavior on purpose.

## Core Concept

The three policies differ on what happens at write time.

| Policy | Write does | Read that misses | Divergence window | Main risk |
|--------|-----------|------------------|-------------------|-----------|
| Write-through | writes DB, then cache | fill from DB | none on the write | write latency, still stale on others |
| Write-back | writes cache, flushes DB later | cache or DB | between flush and write | data loss on crash before flush |
| Cache-aside | writes DB, invalidates cache | fill from DB | brief | race between write and refill |

### Write-through

Every write goes to the database and then immediately to the cache. Reads are always satisfied from the cache, because the cache is guaranteed to be as fresh as the database. The cost is that the write path now has two stops, which adds the cache latency to every write even when nobody reads that key. Write-through is honest but slow on the write, and it does nothing for the shared problem of a value written by a different route, because a second writer that bypasses the cache leaves it stale.

### Write-back

The write goes to the cache first, and the database receives it later, on a flush timer or when the cache evicts the key. The throughput is the best of the three because the write path is one stop and the database sees amortized batches. The risk is that the cache is now the only copy for a window, and a crash inside that window loses committed-to-cache writes. Write-back is for the systems where the throughput payoff exceeds the acceptable loss.

### Cache-aside

The read path fills the cache on a miss and reads the database otherwise. The write path updates the database and deletes the cached key. That delete is the subtle part: on the next read, the miss refills from the new database state. Cache-aside is the default and the correct default for most applications, because it never has to keep the cache and database in lockstep, it only needs to avoid a stale read between the delete and the refill.

Diagram: read and write flow of the three cache policies.

<svg width="900" height="470" viewBox="0 0 900 470" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto">
      <path d="M0,0 L7,3 L0,6 Z" fill="#222"/>
    </marker>
  </defs>

  <text x="30" y="28" font-family="Arial" font-size="15" font-weight="bold" fill="#222">Write-through</text>
  <rect x="30" y="48" width="90" height="40" rx="6" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="75" y="72" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Write</text>

  <rect x="180" y="48" width="90" height="40" rx="6" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="225" y="66" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Cache</text>
  <text x="225" y="82" text-anchor="middle" font-family="Arial" font-size="12" fill="#555">first</text>

  <rect x="330" y="48" width="90" height="40" rx="6" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="375" y="72" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">DB</text>

  <path d="M 122 68 L 178 68" stroke="#222" stroke-width="2" fill="none" marker-end="url(#arrow)"/>
  <path d="M 272 68 L 328 68" stroke="#222" stroke-width="2" fill="none" marker-end="url(#arrow)"/>

  <text x="30" y="138" font-family="Arial" font-size="15" font-weight="bold" fill="#222">Write-back</text>
  <rect x="30" y="158" width="90" height="40" rx="6" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="75" y="182" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Write</text>

  <rect x="180" y="158" width="90" height="40" rx="6" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="225" y="176" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Cache</text>
  <text x="225" y="192" text-anchor="middle" font-family="Arial" font-size="12" fill="#555">holds write</text>

  <rect x="330" y="158" width="90" height="40" rx="6" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="375" y="182" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">DB</text>

  <path d="M 122 178 L 178 178" stroke="#222" stroke-width="2" fill="none" marker-end="url(#arrow)"/>
  <path d="M 272 178 L 328 178" stroke="#222" stroke-width="2" stroke-dasharray="6 4" fill="none" marker-end="url(#arrow)"/>
  <text x="300" y="168" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">flush later</text>

  <text x="470" y="28" font-family="Arial" font-size="15" font-weight="bold" fill="#222">Cache-aside</text>

  <text x="470" y="52" font-family="Arial" font-size="13" font-weight="bold" fill="#333">Read</text>
  <rect x="470" y="64" width="90" height="40" rx="6" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="515" y="88" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Get key</text>

  <rect x="620" y="64" width="90" height="40" rx="6" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="665" y="88" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Cache</text>

  <rect x="770" y="64" width="90" height="40" rx="6" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="815" y="88" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">DB</text>

  <path d="M 562 84 L 618 84" stroke="#222" stroke-width="2" fill="none" marker-end="url(#arrow)"/>
  <text x="590" y="74" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">miss</text>

  <path d="M 712 84 L 768 84" stroke="#222" stroke-width="2" fill="none" marker-end="url(#arrow)"/>
  <text x="740" y="74" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">fetch</text>

  <path d="M 815 106 C 815 130 665 130 665 106" stroke="#222" stroke-width="2" stroke-dasharray="6 4" fill="none" marker-end="url(#arrow)"/>
  <text x="750" y="128" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">fill cache</text>

  <text x="470" y="178" font-family="Arial" font-size="13" font-weight="bold" fill="#333">Write</text>
  <rect x="470" y="190" width="90" height="40" rx="6" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="515" y="214" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Write DB</text>

  <rect x="620" y="190" width="90" height="40" rx="6" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="665" y="208" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Cache</text>
  <text x="665" y="224" text-anchor="middle" font-family="Arial" font-size="12" fill="#555">delete key</text>

  <path d="M 562 210 L 618 210" stroke="#222" stroke-width="2" fill="none" marker-end="url(#arrow)"/>
  <text x="590" y="202" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">then</text>

  <text x="30" y="270" font-family="Arial" font-size="15" font-weight="bold" fill="#222">When each fits</text>
  <text x="30" y="294" font-family="Arial" font-size="13" fill="#444">Write-through: writes are rare, reads must be exact.</text>
  <text x="30" y="314" font-family="Arial" font-size="13" fill="#444">Write-back: writes are frequent, loss is acceptable.</text>
  <text x="30" y="334" font-family="Arial" font-size="13" fill="#444">Cache-aside: the default, delete key on write, refill on miss.</text>
</svg>

### What the table hides

The policies do not just differ in ordering; they differ in the type of inconsistency they tolerate. Write-through still goes stale when a second writer writes to the database without the cache, which is the common case in any system with two writer paths. Write-back accepts the full window of "cache has it, database does not," and the crash window is real. Cache-aside's failure is a specific race: a write deletes the key, then another writer writes a new value, and a concurrent reader misses, loads the old database row into the cache, and the cache now holds a value older than the just-committed one. That race is why cache-aside pairs with a TTL as a backstop.

### The default is cache-aside

For most applications the answer is cache-aside: database owns the truth, cache is a best-effort second copy, writes invalidate, reads fill. It is the policy with the fewest moving parts, and its failure mode, a transient stale read, is the cheapest to tolerate. Write-through is for the write path you want to guarantee fresh; write-back is for the write storm where batching buys the throughput. Naming the policy, and the window it tolerates, is the whole assignment.

## Real Production Usage

Spring's `@Cacheable` with a `CacheManager` is cache-aside in one annotation, and it only works if the paired `@CacheEvict` fires on the mutating method. Redis as the cache manager makes the read-miss-fill loop explicit. Write-back appears in systems where a local cache like Caffeine batches writes before a scheduled flush to a slower store. The production discipline is knowing which annotation half you wrote.

## Common Mistakes

1. **Updating the cache instead of invalidating it.** An update that races with another writer leaves the cache stale and the database correct, and you cannot tell which is true.
2. **Choosing write-back where a crash loses money.** The flush window is the loss window, and a payments workload is the wrong place to trade it.
3. **Ignoring the cache-aside race.** Without a TTL backstop, a racing refill can permanently lodge a stale value in the cache.

## Interview Perspective

Interviewers ask you to compare the policies and then to pick one for a workload. Weak: "write-through is safer." Strong: "cache-aside is the default because it deletes on write and refills on read, and I would take write-back only when batching pays for the loss window, never for money." They often probe the cache-aside race specifically to see if you know invalidation beats update.

Follow-up: "What happens if the delete in cache-aside fails?" They want you to talk about the TTL backstop and the bounded staleness window.

## Knowledge Check

1. A write goes to the cache first and the database later. Name the policy, then name the disaster if the process dies before the flush.
2. Cache-aside deletes on write instead of updating. Why is the delete better than the update in the presence of a racing reader?
3. You pick write-through for a ledger. What does that cost on the write path, and what staleness does it still not protect you from?

## Key Takeaways

- Write-through, write-back, and cache-aside differ in the truth order and in the divergence window each tolerates.
- Cache-aside is the default: DB owns truth, reads fill, writes delete, TTL backstops the race.
- Choose write-back only when batching pays for a bounded loss window; never for money movement.

## What's Next

The right policy in hand, the remaining problem is that a cache is one tool among several, and the next article steps back to the whole persistence design. Persistence best practices is where the chapter's scattered decisions, fetch strategies, locks, transactions, and caches, get reconciled into a checklist you can apply to a real schema.

---

This article explains write-through, write-back, and cache-aside by their write ordering and the inconsistency window each tolerates. It argues that cache-aside is the correct default because it invalidates on write rather than updating, and that write-back's crash window makes it unfit for money-bearing workloads.