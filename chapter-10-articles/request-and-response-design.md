# Request and Response Design

## Learning Objectives

- Design the request and the response as one contract, and know what each side is allowed to assume.
- Use status codes and headers to answer what the success case returns, where a created resource lives, and how the caller can proceed.
- Apply the two rules of response shape: never return what the caller did not ask for, and never make the caller ask twice.

## Introduction

Every API call is a pair. The caller sends a request, the server sends a response, and the two have to hold together as one agreement. The request says what the caller wants; the response says what happened, with the status, the headers, and the body all carrying part of the answer. Design the pair together, because a request designed against a response nobody can parse is a contract that fails on its first real user.

## Problem Statement

The failure mode here is the response that leaves the caller guessing. A `POST /orders` that returns `200 OK` with no body, so the caller does not know the order id, whether it was created or updated, or where to look next. A `GET` that returns a body, but the client has to parse the presence of fields to figure out what kind of thing it got. And the request side of the pair: the caller cannot tell, from the docs, what is required, what defaults, and what happens when they leave something out.

That vagueness is not cosmetic. It is where integration bugs live. A caller assumes the order id is in the body; the API returned it in a header. A caller assumes `POST` returns the created object; the API returns just the id. Every ambiguity in the pair is a guess, and a guess is a future incident.

## Core Concept

### The request carries the intent

A well-designed request tells the server three things: the target, the intent, and the payload. The target and the intent come from the path and the verb, both settled in the REST article. The payload is what the caller hands over, and the design questions are about what goes in the body versus what goes in the path or a header.

The body is where the payload lives for a create or update. It should be the representation of what the caller wants the server to end up with, not a pile of incidental flags. A request to create an order carries the customer, the lines, the shipping address. The `customerId` lives in the body or the path depending on how you modeled the resource, but whatever the choice, it should be stable across the API.

Query parameters carry the filters and paging for a list, and they should be optional with sane defaults. The rule of thumb: the body for the thing, the query for the knobs, the path for the identity. Keep the knobs small, because every query parameter is a dimension of behavior a caller can get wrong.

### The status code is the first sentence of the answer

The response's job is to be unambiguous about what happened. The status code is the headline. Use the standard codes and use them correctly, because callers and client libraries branch on them. `200` for a successful read, `201` for a created resource, `204` for a successful delete that returns nothing. `400` for a malformed or invalid request, `401` when the caller is not authenticated, `403` when they are but are not allowed, `404` for a resource that does not exist, `409` for a conflict with the current state, and `429` when they are being rate-limited.

Most of the design work is making the status code mean exactly one thing. The classic confusion: `200` on a create instead of `201`, which tells the caller nothing about whether the resource is new or was already there. And a `400` used for "the caller is not logged in", which the client then cannot distinguish from a malformed payload. The code is the contract's strongest signal; keep it precise.

### The headers finish the sentence

The status code says what happened; the headers say what to do next. The two that matter most are `Location` and the caching headers.

On a `201 Created`, return a `Location` header with the URL of the new resource. That single header makes the create self-describing: the caller can hand the response to `GET` and retrieve the thing it just made. It is the difference between "the call worked" and "here is where the result lives."

```java
@PostMapping
public ResponseEntity<OrderDto> create(@RequestBody CreateOrderRequest request) {
    OrderDto created = orders.create(request);
    URI location = URI.create("/orders/" + created.id());
    return ResponseEntity.created(location).body(created);
}
```

`ResponseEntity.created()` sets the `201` and the `Location` header in one line. The response also returns the created representation, which is the second rule of response design: a create returns the thing it made, so the caller does not have to call `GET` again to learn what the server assigned. Cache headers, `Cache-Control` and `ETag`, matter more in the HTTP article later, but the short version is they let correct responses be cached, which is most of the value of a read API.

### The response body is a promise

The body is what the caller relies on, so its shape is a promise about the future. Two rules.

First, return what the caller asked for and not everything you have. A `GET /orders/{id}` should return the order the caller can act on, not the internal audit log, the tenant id, the ORM version column. Less is more here, because every extra field is a promise you have to keep. The `@JsonIgnore` pattern from the DTO article is how you trim it.

Second, never make the caller ask twice. The response to a create returns the created representation. A response to an update returns the new state, not just a code. The cost of asking again is an extra round trip and an extra chance to break; the server already has the object in hand, so return it.

### Designing the pair together

The concrete test of a request-response pair is whether a stranger can write the client from the docs alone. Write the request, write the response, and then check each field: is every required field required, is every default stated, is the response the thing the caller needed to proceed? The pairs that pass this test tend to look the same: explicit methods, exact status codes, a `Location` on create, and a body that is the answer, not a hint.

## Real Production Usage

Stripe and the platform APIs are the reference for this. Create a payment and you get a `201` with the `Location` and the full payment object, so a client that just wants to show the confirmation needs one call, not two. GitHub returns `201` on a created issue with the issue object and a `Location` to its URL. The pattern is so consistent across the big platforms that it is effectively a convention: create returns `201`, `Location`, and the representation.

In Spring, `ResponseEntity` is the tool for the whole pair. `created()`, `ok()`, `noContent()`, and `status()` cover the standard responses, and the controller never returns a bare object when the response has to carry headers. Returning a plain `OrderDto` from a `@PostMapping` and losing the `Location` is the small mistake that starts the drift toward the unparseable contract.

## Common Mistakes

**`200` on everything.** The controller returns `200 OK` for creates, for deletes, for every path. The caller cannot tell create from update from delete, and client code has to sniff the body to know what happened. Use the codes; they are free and they are the signal.

**The create that returns nothing.** A `POST` that returns `201` with an empty body or no `Location`, leaving the caller without the id of what it just made. The caller then re-queries by some other key, or stores a client-generated id that the server ignored. Return the thing and its location.

**The response that leaks the store.** A `GET` returning the internal representation, version fields, audit trails, lazy proxies serialized, because the controller handed the entity to Jackson. Trim to the shape the caller needs; the DTO article covered how.

## Interview Perspective

Interviewers who ask about request and response design are checking whether you treat the exchange as a designed pair or as whatever the framework returns. A weak answer describes "the request object and the response object." A strong answer names the pieces: the method and path carry the intent, the status code is the headline, the `Location` and caching headers say what to do next, and the body is a promise about what the caller can rely on.

The follow-up that sorts people is "you just created an order. What does the response contain?" The strong answer is `201 Created`, a `Location` header pointing at the new order, and the created representation, so the client never has to guess or re-query. The candidate who says "200 and the object" is missing the `Location` and the code semantics, which is the tell of someone who has not run a real integration.

Common follow-ups:

- "When do you return `204` versus `200`, and what goes in the body of each?"
- "A create is not idempotent and the caller retried. How does the response help them tell a retry from a new create?" (Preview: idempotency is a later article.)

## Knowledge Check

1. A `POST /orders` returns `200 OK` with no body. List everything the caller still does not know, and the exact response that would fix it.
2. Why does a successful create return `201` with a `Location` and the representation, instead of `200` with a message, and what does that do for the client that calls `GET` next?
3. A `GET /orders/{id}` returns the entity with version columns and lazy proxies. Name two promises this response makes to callers that you now have to keep.

## Key Takeaways

- The request and the response are one contract; design them together.
- The status code is the headline: `201` on create, `Location` for where it lives, `204` for an empty success.
- A create returns the thing it made so the caller does not ask twice.
- The body is a promise: return what the caller needs and nothing the caller did not ask for.

## What's Next

The pair holds together, and the next question is what happens when the request is wrong. The next article is about input validation, which is where the request meets its limits: which fields are required, what a value may be, and where that checking lives so a bad request is rejected before it touches the system.

---

This article explains request and response design as one contract, the status code, the Location and caching headers, and the body answering together. It argues a create returns 201 with the Location, and a reply that leaves you guessing fails its first user.