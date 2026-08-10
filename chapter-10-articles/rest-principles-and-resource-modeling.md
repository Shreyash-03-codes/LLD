# REST Principles and Resource Modeling

## Learning Objectives

- Model a problem domain as a set of resources, and name the difference between a collection and a single item in the path.
- Map the standard HTTP verbs to operations so the verb carries the intent and the URL names the resource.
- Recognize statelessness as the property that lets a REST API scale and retry, and state what it gives up in exchange.

## Introduction

REST is how most of this industry has agreed to put an API over HTTP. The core idea is smaller than people make it: you model the world as resources, nouns, and you act on those resources with a fixed vocabulary of verbs. The URL names what thing, the verb says what you want to do to it, and the state of the caller lives with the caller, not on the server.

That framing is worth holding onto because it is the exact opposite of how a lot of turned out. The API as a bag of procedure calls, `/doThing`, `/saveOrder`, is RPC wearing HTTP, and it throws away the one thing REST buys you: a shape that is guessable by a human and consistent for a machine.

## Problem Statement

Without resource modeling, APIs drift into RPC. The endpoint names are verbs describing operations: `POST /charge`, `POST /get-work-items`, `POST /cancelOrder`. Every operation invented its own endpoint, the path is a function name, and the client has to know the full procedure vocabulary to talk to you.

RPC endpoints fail you in a specific way. They do not compose. There is no way to guess the collection of orders by paging, you have to call whatever `getOrders` the original author thought to add. Every new operation is a new endpoint with a new name, the surface grows by accretion, and nothing is predictable. A caller that wants to list, filter, and page through orders has to hope each of those operations exists as its own method.

The second failure is the misplaced verb. A `POST /orders/123/update` that changes the order and returns nothing, or a `GET` that happily deletes because the path contained `delete`. When the verb does not carry the intent, the URL is doing double duty, and the behavior is scattered all over the contract in a way no one can reason about.

## Core Concept

### Resources are the nouns

The first move is linguistic: decide what things exist in your domain, and those become resources. Customers, orders, invoices, shipments. A resource is a thing you can say "give me one of" or "change this one", so it maps to a noun with an identity. In the URL, a resource appears in two forms: a collection, `/orders`, meaning the whole set, and a single item, `/orders/{orderId}`, meaning one of them.

The collection form is where the verbs do the standard work. Get the whole set, list the ones that match, and create a new one. The item form is where you read, replace, or delete the one. This split is the backbone of the whole REST model, because it lets a client reason about any resource by the same rule.

```java
GET    /orders          -> list the orders
POST   /orders          -> create a new order
GET    /orders/{id}     -> fetch one order
PUT    /orders/{id}     -> replace one order (or the field you update)
DELETE /orders/{id}     -> remove one order
```

Read that and it carries its own weight. Every line is the same template, and a client that knows `orders` and `{id}` can predict the whole set without documentation. That predictability is the resource model paying for itself.

### The verbs carry the intent

The verbs are not decoration, they define what you are allowed to do, and the URL stays a noun. GET asks for a representation, and must not change server state, which is why a correct GET is safe and re-runnable. POST creates a new thing and is the only one the verb does not offer a safe retry, which is the idempotency discussion you will see later in the chapter. PUT replaces the whole thing. DELETE removes it.

The consequence of assigning intent to the verb is that you stop naming operations. "Update", "list", "remove" are not URL fragments, they are the verbs. The URL is only ever `thing` or `thing/{id}`. That rule, url is a noun, verb is the verb, turns the whole API into one learnable pattern instead of a directory of one-off URIs.

```java
@RestController
@RequestMapping("/orders")
class OrderController {

    @GetMapping
    List<OrderDto> list() { ... }

    @GetMapping("/{id}")
    OrderDto find(@PathVariable Long id) { ... }

    @PostMapping
    OrderDto create(@RequestBody CreateOrderRequest request) { ... }
}
```

The Spring mapping says it plainly: the path is the noun, the annotation is the verb.

Diagram: the resource hierarchy and its verbs

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 400" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#33475b"/>
    </marker>
  </defs>

  <text x="180" y="82" text-anchor="end" font-size="12" fill="#5a6b7a">collection</text>
  <rect x="320" y="46" width="300" height="72" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="470" y="80" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">/customers</text>
  <text x="470" y="105" text-anchor="middle" font-size="11" fill="#5a6b7a">GET  ·  POST</text>

  <line x1="470" y1="118" x2="470" y2="168" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <text x="180" y="212" text-anchor="end" font-size="12" fill="#5a6b7a">item</text>
  <rect x="320" y="172" width="300" height="72" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="470" y="206" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">/customers/{id}</text>
  <text x="470" y="231" text-anchor="middle" font-size="11" fill="#5a6b7a">GET  ·  PUT  ·  DELETE</text>

  <line x1="470" y1="244" x2="470" y2="294" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <text x="180" y="338" text-anchor="end" font-size="12" fill="#5a6b7a">collection</text>
  <rect x="320" y="298" width="300" height="72" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="470" y="332" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">/customers/{id}/orders</text>
  <text x="470" y="357" text-anchor="middle" font-size="11" fill="#5a6b7a">GET  ·  POST</text>
</svg>
```

The vertical chain is the point: a collection lets you list and create, its item lets you read, one replace, and remove, and the nested path reuses the same collection-and-item rhythm one level down. The verb set repeats at every level, which is what makes an unfamiliar resource predictable.

### The statelessness restriction

True, a REST API is stateless, the server holds no client conversation. Every request carries everything needed to answer it, usually the auth token and any state in the URL, and the server does not remember what the previous request was. This is the property that lets you route any request to any replica of the server, scale out, retry a failed request, and run the backend behind a load balancer without maintaining per-client memory.

Statefulness is what you are giving up. There is no session, no "you were browsing, let me continue where you left." A caller that needs to track a dozen selected orders keeps that list client-side, or sends it, every request. That coordination is a real cost, and the exchange is the freedom to scale and the ability for the caller to resume any request. For most modern APIs the trade is right, and the article that closes this chapter will revisit it.

## Real Production Usage

REST is the default for these, and the platform level is the biggest articulation of the model. Think of the rate limits on anonymous and how a client that lists repositories, creates an issue, and pages through a search result, is following exactly this resource shape. Spring, and Spring Data REST, produce these by convention, which is both the blessing and the trap.

The blessing is that `@RestController`, Spring HATEOAS, and `ResponseEntity` make the noun-and-verb pattern cheap to build. The trap is the same reason it is cheap: serializing a domain object and slapping a `@GetMapping` does not make it a REST API, it makes it a noun-shaped portal to your tables. The resource model is a design decision about what your nouns are and what the client can do to them, and no amount of annotation recovers it if the noun choice was the wrong mirror of your database.

## Common Mistakes

**Treating actions as resources is a mistake, but verbs like `generate`, `align`, `convert`, `confirm` are a smell in the URL.** When a bank needs a transfer, do not model `POST /transfer` as the resource thing. A transfer that creates a movement can be `POST /transfers`, a resource with its own id, and the completed fact later read from `GET /transfers/{id}`. Verbs in the path are a sign you are describing a procedure and not a resource.

**The form-data semantics of PUT.** Many teams use PUT to do partial updates because it is convenient, but PUT means replace the whole item. A partial patch is `PATCH` when you must. Conflating them makes a `PUT` that only updates one field a lie, and a caller that expects replace semantics gets a surprise it cannot recover from.

**Designing the resource by the table.** Modeling your API classes straight off the database entities, passing the `User`, `id`, `Role` cross-references up to the client, is the resource model confused with a schema. The client should see designed nouns, often the domain shape, not the storage shape. Table in the model, not the resource.

## Interview Perspective

Interviewers use REST as a test of whether you can name the shape of a contract more than whether you know the spec. A weak answer rattles off "GET, POST, PUT, DELETE." A strong answer talks about resources as nouns, the verb as the operation, the collection and the item, and what statelessness buys you and what it costs.

The follow-up that sorts people is "model an order with line items as a REST API." The strong answer names the resources (`orders`, an order being the aggregate root), where the lines live (`/orders/{id}`, you rarely need them at top level), and the verbs that act on each. The candidate that starts naming service methods instead of nouns is thinking in RPC, and that is the tell.

Common follow-ups:

- "Is `POST /orders/{id}/approve` a good design, and why not?"
- "What is the difference between PUT and PATCH, and when do you use each?"

## Knowledge Check

1. `POST /cancelOrder` and `POST /orders/{id}/cancel`. Both express a cancel. Why is one better than the other, and what resource does the better one expose?
2. Your server must hold nothing about the caller between requests, yet a caller wants to page through 300 orders. Where does the page state live, and why does the server get to drop it?
3. Why is GET required to have no side effects, and what breaks the moment a `GET` starts mutating state, in terms of caching and retry?

## Key Takeaways

- The URL is a noun (a resource); the verb is the operation on it.
- A resource appears as a collection (`/orders`) and a sample (`/orders/{id}`), and each gets its predictable set of verbs.
- Stateless servers do not carry conversation state, which is what lets them scale and retry and run behind a load balancer.
- Design the nouns that make sense to the client; the entities in the DB are a candidate, not the final answer.

## What's Next

The nouns and verbs set up the next problem: what, exactly, travels over the wire. The next article is about DTOs and mapping, which is where the resource model meets a concrete shape, why the JSON you send is almost never the domain object, and how MapStruct and Jackson turn the domain into a stable transport shape without you hand-writing the conversion.

---

This article explains REST as resources and verbs, the URL the noun, the method the operation, a token that stays guessable. It argues verbs in the path are RPC in a REST disguise, and statelessness is the price of scale and retryability.