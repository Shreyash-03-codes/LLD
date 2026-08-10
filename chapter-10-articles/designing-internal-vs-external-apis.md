# Designing Internal vs External APIs

## Learning Objectives

- Split the contract your own services consume from the one the public consumes, and design each for its own audience.
- Weigh velocity against stability for each side on its own terms, so change is cheap where you own the caller.
- Keep one domain core and two thin adapters, so the two DTOs diverge without the application logic forking.

## Introduction

The word "API" hides a fork in the road. An interface your frontend calls so it can ship today, and an interface your partners will consume for years, are not the same product. One wants to move at your team's pace, the other must hold still for outside callers you cannot force to upgrade. The internal contract and the external contract have different incentives, different lifetimes, and different shapes.

This, the last article of the chapter, ties the advice together: how to give a service a face for the inside and a face for the outside, and why the two should not be one.

## Problem Statement

The failure is designing one API and serving both audiences from it. A single public surface hit by your frontend and your third-party partners, and the two grind against each other. The frontend needs to ship a new field of the week; the partners need utter stability. One surface freezes the frontend or breaks the partners, and nobody can win.

The same conflation costs you on identity. The internal caller is your own service, which can be trusted with a key and a renewing lease. The external caller needs a scoped token and an audit trail. One API doing both either makes the public side unsafe or burdens the internal side with ceremony it does not need. Neither reads right.

## Core Concept

### Two surfaces, one core

The central move is to keep the same domain service and put two thin boundaries in front of it. The internal surface and the external surface are each a controller, a DTO set, and a Mapper, in front of the same service. Exactly two DTOs and two routes, no fork in the logic.

The reason it works is that the two audiences want different shapes. The internal caller, a frontend, a sibling service, wants lean and fast and shared-zone simple, because the people who call it can handle a change overnight. The external caller, a partner, an SDK user, wants documented and stable and versioned, because the contract is the only handle it will ever have. The core does not need to know which kind of caller answered, the adapters do.

### What each side trades

The real difference is a deal about stability. An internal API can afford to change, because its consumers are your own. A deploy can move the client and the server at once, and a breaking change is a coordination problem, not a public crime. So the internal surface optimizes for velocity. It can be lean, barely documented, fast to change, and unversioned, because the one who calls it is the one who writes it.

An external API is the inverse. Its consumers are outside and you do not deploy them, so a breaking change is a permanent injury to a client you may never update. The external surface trades velocity for stability. It is documented, versioned, deprecate-with-a-window, additive-first, everything this chapter argued. When a breaking change touches a public contract, the shape has to hold.

The two columns:

| Besides | Internal | External |
|---------|----------|----------|
| Stability | changes often | long-lived |
| Auth | service key | scoped token |
| Versioning | rarely needed | near-mandatory |
| Shape | lean, purpose-built | rich, documented |
| Change cost | low | high |

Read the internal column and it is about speed; read the external column and it is about survival. That gap is the whole point of this article.

### Two different gates

Security is where the fork is easiest to see. Anything inside the trust zone, the frontend behind the gateway, a sibling service, can authenticate with a service key, a static shared secret, trusting the zone as a perimeter. No OAuth, no scope dance, no ten-minute negotiation of claims, just a key and a request that is trusted because it came from inside.

The external gate is the opposite. Those callers authenticate with a scoped token with claims, a JWT or a scoped API key, and the server authorizes each request down to a narrow claim. This is where OAuth2 earns its living. The external side pays an expensive identity negotiation because it cannot trust the caller, unlike the internal. The trust that was nearly free inside becomes a real cost outside.

### The DTO is the joint

The seam between the two surfaces is the DTO. The internal DTO is lean: the id, the fields the internal caller acts on, and no version. The external DTO is the product: it is the versioned contract, it carries the fields a public consumer needs, and it changes only under a deprecation window, following the versioning article.

Because both are DTOs over the same service, a domain change ripples to two thin transcripts and never into the deep logic. The cost of two surfaces is exactly two DTOs and two Mappers, and the value is that the internal team gets pace while the public callers get stability, at the same time, without the two fighting each other.

Diagram: one core, two adapters

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 420" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#33475b"/>
    </marker>
  </defs>

  <rect x="330" y="40" width="230" height="70" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="445" y="76" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Domain service</text>

  <line x1="430" y1="110" x2="242" y2="238" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="462" y1="110" x2="660" y2="242" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="80" y="240" width="325" height="90" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="242" y="286" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Internal API</text>
  <text x="242" y="310" text-anchor="middle" font-size="11" fill="#5a6b7a">lean, shared zone</text>

  <rect x="500" y="245" width="325" height="90" rx="10" fill="#eef6ee" stroke="#4a8a4a" stroke-width="1.5"/>
  <text x="662" y="291" text-anchor="middle" font-size="13" font-weight="bold" fill="#2f6b2f">External API</text>
  <text x="662" y="315" text-anchor="middle" font-size="11" fill="#4a8a4a">public, stable, versioned</text>
</svg>
```

One domain service, two adapters. The internal one is lean and fast to change; the external one is the stable, published surface. The two DTOs drift apart freely, and the core never needs to know which boundary answered the call.

### Keep the boundaries thin

The trap with two surfaces is building two applications. The weight always sits in the single domain service, and the internal and external are each a controller, a DTO set, and a Mapper. The moment the external layer starts carrying real logic, or the internal sprouts the OAuth the outside needs, the two products have merged and the whole split was for nothing. Identities stay out of the adapters, and the adapters stay thin.

## Real Production Usage

The platform companies are the living example. Their public API is versioned, documented, rate-limited, running deprecations; their internal calls, in the same service, are fast and unversioned, because the same team ships the client and the server on the same deploy. The OAuth is for the outside; the inside shares a key. That two-face shape is what survives.

In Java it usually means two modules sharing the domain jar: one exposing the REST for the public, one the internal contract. The domain is the single artifact, the adapters are the controllers and the DTOs, and the dependency points inward, which is this whole chapter's lesson. The modular split is how a codebase holds the speeds apart without the wires tangling.

## Common Mistakes

**One contract for both.** A single surface the frontend changes and the partners may not, so either the frontend is frozen or the partner is left on a broken contract. Split the surfaces even though the bytes hit the same machine.

**The internal built as a small external.** An internal surface with versioning, deprecation, rate limits, and a scope dance slows you down for a contract your own team wants to change. Make the internal lean.

**The public shape into the domain.** Baking the external's special fields into the service, the entity grows a public shape. Keep the external in the external DTO, the core in the core.

## Interview Perspective

The interviewer wants to see you split the contract by audience. A weak answer says "have one API and lock it down." The strong answer names the fork: internal is fast and trusted and lean, external is stable, versioned, scoped, and both are thin adapters over one core that can change either without merging.

The follow-up that sorts people is "your frontend and your partners hit the same API." The candidate who says the frontend is an internal consumer and the third parties are external, then two surfaces, one fast and one slow, has the shape. The candidate who defends a single API with "it is simpler" has not yet felt the frontend grinding on the partner contract.

Common follow-ups:

- "When is it worth the cost of a second surface just for internal callers?"
- "Where does versioning live in the external DTO, and why not the internal?"

## Knowledge Check

1. A frontend and a partner integration hit the same shape. Name a change the frontend could absorb and one the partner could not, and the surface each wants.
2. The internal caller uses a service key, the external a scoped JWT. Why is that boundary one of trust, and what does it cost the external side?
3. Two DTOs, one service. Where does the version go, and how does the internal DTO change in one deploy without touching any external contract?

## Key Takeaways

- Internal and external are two products; keep one core and two thin adapters.
- The internal leans velocity, the external leans stability and version.
- Key inside, scoped token outside: trust is cheap in the zone and dear beyond it.
- The versioned surface is external, and the DTO is the seam where they stay apart.

## What's Next

This is the last article of the API and Interface Design chapter, and the shape the next turn depends on it: the boundary now faces outward, and a persistent store is the ground behind it. The next chapter is Persistence and Data Design, which turns the design to the database. It takes the domain model and the contract you just built and decides the tables, the keys, the transactions, and the consistency that keep them durable and fast, and where the JSON you designed gets its long-term home. The change is direction, the API chapter faced the caller, the persistence chapter faces the store.

---

This article explains the internal and external API as two contracts, a lean fast surface for your own services and a stable, versioned one for the world. It argues that one core with two thin adapters is right, and one contract for both breaks either.