# API Versioning Strategies

## Learning Objectives

- Choose the carrier for the version, the URL path or a header, and name the cost of each.
- Run a new version alongside the old one and sequence the handoff so no caller is broken with no warning.
- Keep most growth additive, and reserve a new version for the shape that cannot be added to.

## Introduction

At some point the contract you shipped cannot bend into what you need. You want to change the meaning of a field, remove one, or split a resource. Every caller you shipped to has built against the old shape, so you cannot just delete it. Versioning is how you run two shapes at once: the old one keeps the existing callers happy, the new one carries the change.

Versioning is not the first tool. It is the last one, because carrying two contracts is real cost. The design goal is to stay additive as long as possible and to version only when adding would distort the shape more than shipping a second number.

## Problem Statement

An unversioned API that changes on the one original contract breaks every caller at once. Remove a field and the client that reads it now gets a surprise, a null, or a parse error at a time you do not control. There is no migration window, because the same endpoint served both shapes and you flipped it all at once.

The developers that remain end up in one of two places. Either they never change anything and add endless optional fields, accretion that is versioning with a worse name, or they change the endpoint under the callers and eat the breakage repeatedly. Both are versioning done badly. The point of real versioning is to make the coexistence explicit instead of accidental.

## Core Concept

### Additive is the cheapest version

The best versioning is barely needing it. A great many changes are additive: an optional new field, a new endpoint, a new value in an existing enumeration. None of these break a caller on the old shape, so a single version keeps serving everyone for a long time. The rule is additive first: if the change can be expressed by adding an optional field, ship it in the current version.

The line you cross is when a field must change meaning, or the old shape has to disappear. That is where the new version buys its keep. But most of the growth should happen on the additive side, and the versioning event should be rare.

### Where the version rides

The version has to travel on a request, and there are three places it can. Each has a different cost.

The URL version, `/v1/orders/1`, is the most visible and the most common. The client's log line says exactly which shape it got, the version is easy to route, and the infrastructure, a cache, an LB path, can use it. The downside is that the resource tree is duplicated and a caller anchored to `/v1` stays glued to the old. But that "glued" property is also its strength, it is a name that cannot be mistaken.

The header version, a custom `Accept-Version: 2` or the content-negotiation `Accept: application/vnd.orders.v2+json`, keeps the URL clean. The cost is invisibility: the routing is not a trivial URL match, a proxy or a cache counts the URL and not the header, and a client that forgets its version may silently get the wrong one. It is better than it looks though, because a resource can move between versions without the URL changing.

The body version, a version number in the JSON, is close to useless. The routing cannot happen before the body is read, the version is undiscoverable in a log cache, and two callers with the same URL and different bodies get different behavior. For most APIs the realistic answer is a URL number, with the header viable for parts that want the resource tree to stay shared.

### Two versions, one namespace

The machinery of coexistence in Spring is two controllers and two DTOs, and nothing clever:

```java
@RestController
@RequestMapping("/api/v1/orders")
class OrderV1Controller { ... }

@RestController
@RequestMapping("/api/v2/orders")
class OrderV2Controller { ... }
```

The same domain behind both, same service, and the only difference is the DTO and the route. A v1 caller stays on the old shape until it moves; a v2 caller gets the new. The version difference is a mapped difference in the response, which is exactly what the DTO article said a version was. When a field changes meaning, v1 and v2 widens in that DTO and nowhere else.

### The life cycle of a version

A version walks through stages: it ships, it only adds for a while, then a successor ships alongside it, then the old one is deprecated and retired.

Diagram: the stages a version passes through

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 300" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#33475b"/>
    </marker>
  </defs>

  <rect x="20" y="90" width="150" height="80" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="95" y="122" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">V1 ships</text>
  <text x="95" y="146" text-anchor="middle" font-size="11" fill="#5a6b7a">first contract</text>

  <rect x="196" y="90" width="150" height="80" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="271" y="122" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Additive only</text>
  <text x="271" y="146" text-anchor="middle" font-size="11" fill="#5a6b7a">new fields, no breaks</text>

  <rect x="372" y="90" width="150" height="80" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="447" y="122" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">V2 coexists</text>
  <text x="447" y="146" text-anchor="middle" font-size="11" fill="#5a6b7a">both shapes serve</text>

  <rect x="548" y="90" width="150" height="80" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="623" y="122" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">V1 deprecated</text>
  <text x="623" y="146" text-anchor="middle" font-size="11" fill="#5a6b7a">callers told to leave</text>

  <rect x="724" y="90" width="150" height="80" rx="10" fill="#eef6ee" stroke="#4a8a4a" stroke-width="1.5"/>
  <text x="799" y="122" text-anchor="middle" font-size="12" font-weight="bold" fill="#2f6b2f">V1 sunset</text>
  <text x="799" y="146" text-anchor="middle" font-size="11" fill="#4a8a4a">old shape retired</text>

  <line x1="172" y1="128" x2="194" y2="128" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="348" y1="128" x2="370" y2="128" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="524" y1="128" x2="546" y2="128" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="700" y1="128" x2="722" y2="128" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
</svg>
```

The version adds, then a successor appears, then the old is deprecated with a date, and finally retired. The middle is the crucial part: deprecation is a long-running signal to the caller, not a one-time announcement.

### Deprecation is a promise with a date

Deprecation is not a badge you slap on when you are tired of v1. It is a behavior. You tell the callers the version is going away, you give them a date, and you keep serving until then. A warning header on each response and a migration guide are the honest kind. Retiring without that path is a broken caller, which is the one outcome versioning exists to avoid.

The consistent phrase to remember: deprecate, migrate, then sunset. The order is fixed, and the middle step is the one most teams rush.

### When to reach for it

A new version is justified when the shape genuinely cannot be expressed additively: a field has to stop being sent, an existing field changes meaning, a resource splits. That bar is high on purpose. When adding a field would meaningfully distort, the cost of a second version, two DTOs, two routes, two lifetimes, is less than the cost of the lie additive would tell.

## Real Production Usage

Every platform that survived a large ecosystem hit this. The lifecycle you read about, `v1` then `v2` in the path, is the visible scar from that, and it reads as the practical standard. Pretty much every large consumer API keeps the URL number, because it is the one version mechanism a log and a cache can read, and it makes a client's version explicit in every line.

The Java small thing is how cheap the coexistence is: two controllers, two DTOs, one service, and the annotated routes. The actual versioned difference is concentrated in two DTOs, which is the same boundary as the DTO article, now expressed as a pair. Teams that keep it contained, isolating the difference to the DTO and the route, run the pair comfortably for the whole deprecation.

## Common Mistakes

**An additive that is secretly breaking.** A field that changes type, a string to an object, breaks the parse, even though it was "additive" in the log. Additive means the old caller still sees the old shape, not "we did not add a new number."

**Relying on a silent header.** A client sends `Accept-Version: 2` and the server ignores it, so the two of you are on versions no one stated. If you accept a header, honor it and answer an error when not.

**Naming the version, and then retiring before moving.** A deprecation without a date and a path means the callers stay and the sunset becomes the accident. Deprecation is a date and a migration, and both have to exist.

## Interview Perspective

The interviewer is checking whether you reach for the version at the right time. A weak answer "we bump the URL." The strong answer names: additive first, the high bar for a real version, the choice of URL versus header as a practical, the two-route coexistence, and the deprecation with a date as the discipline. The candidate who treats versioning as "change the URL and everyone gets the new thing" has missed the point that the old callers keep running.

The follow-up with the sharpest edge is "a caller is stuck on v1, how do you get it to v2?" The strong answer is the dated migration during deprecation, with the old kept serving until the caller is off. The candidate who says "just cut it over" has confused versioning with a rename.

Common follow-ups:

- "When is a header version better than a URL version?"
- "How is a breaking change different from an additive one?"
- "Your v1 caller ignores the sunset. What do you do?"

## Knowledge Check

1. Adding an optional field is not a version, but changing the meaning of an existing field is. Which side of the line does each fall on, and why?
2. Which version carrier, URL or header, tells a cache or a log which version it got, and why does a header hide?
3. Two DTOs, two routes, one service. Why is that the whole machinery of versioning, and where does the actual difference live?

## Key Takeaways

- Additive-first keeps a single version alive longer; the new version is for the shape that cannot extend.
- The URL version is visible and routable, the header version keeps the tree clean and hides.
- The versioned pair is two controllers and two DTOs over the same service.
- Deprecate with a date and a path; sunset only after the last caller moved.

## What's Next

Versioning lets a caller aim at the same shelf even as you change it; the next article is about the fact that the caller may call twice. Idempotency in APIs covers how a server tells a retry of something it already did from a brand-new command, with an idempotency key in the header and the store-and-reply behavior on top.

---

This article explains API versioning as carrying two shapes at once and retiring the old contract with a dated deprecation. It argues that additive-first takes you far, and a deprecation without a date is only a threat.