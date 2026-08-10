# Error Handling and Standardized Error Responses

## Learning Objectives

- Design one error envelope and return it for every failure, so a client can branch on it reliably.
- Centralize error handling in a single global handler instead of scattering `try/catch` in every method.
- Pick each component of the envelope, the HTTP status, a stable code, a message, and details, for what each one is for.

## Introduction

Every API fails, and the failure is part of the contract. A client does not stop calling you when you have errors, it must handle them, and it can only handle them well if the failures look like the same shape. Standardized error handling is what makes "the request failed" an actionable piece of information instead of a mystery the caller has to reverse engineer.

## Problem Statement

The failure is inconsistency. Each endpoint answers an error differently. One returns a raw `404` with HTML from the server, another returns a JSON object with a message, a third returns a code on one key and a message on another. The client that wants to surface "this order was already paid" has to know which endpoint it called and which shape to expect.

But miscoded errors break automation too. A `500` with an internal exception message leaks stack and can expose internals, or a `500` is returned for a validation problem that should have been a `400`, so the caller cannot tell a client mistake from a server bug. And without a stable identifier, there is no way to match the client's complaint, "it failed", to the server's log line that caused it. That traceability gap turns every incident into detective work.

## Core Concept

### One envelope for everything

The first move is to decide the shape, and make it the shape of every error response. One record, one JSON structure, for validation failures, not found, forbidden, and internal errors alike.

A workable envelope:

```json
{
  "code": "ORDER_ALREADY_PAID",
  "message": "Order 7841 was already paid and cannot be cancelled.",
  "details": ["field=cancelReason problem=requested"],
  "traceId": "8f2c0e1-3d1c-49ab-bc12-9f00a1c2b1"
}
```

The four fields each do one job. `code` is a stable machine identifier, a short uppercase string, documented, and the thing a client branches on. Crucially, the code does not change, even when the human `message` text is reworded, so a client that matches on `ORDER_ALREADY_PAID` keeps working as copy improves. `message` is for a human, short and specific to this failure. `details` is optional, a list of per-field problems for validation errors. `traceId` connects the response to the server's log.

### The status code is not the error code

A common confusion: the HTTP status and the application error code are two different contract fields. The status is the coarse class, `400` for bad request, `404` for missing, `409` for conflict. The error code is the precise reason, `ORDER_ALREADY_PAID`. Inevitably, two distinct reasons share a status, which is exactly why the error code exists: the caller branches on it, and the status tells only the coarse class.

So the mapping is: a whole class of failures share a status, and each carries its own code in the body. The set of statuses is small, which reduces the client's job to reading `code` and acting, reliably, where scraping `message` is a bug farm.

### Choosing a contract

Naming the error codes is a small piece of design that most teams skip. The code is a string identifier, so it should get the same discipline as a response field. A code like `ORDER_ALREADY_PAID` is good: human-readable when a colleague reads a log, stable, long enough to be unique. A code like `1003` or `E_4` carries the same information to a machine but is worse for the human who greps logs. The codes that survive are long `SCREAMING_SNAKE_CASE` phrases: `ORDER_ALREADY_PAID`, `USER_NOT_FOUND`, `RATE_LIMITED`. It is a category, never a stack trace, and it does not carry a value.

A code is a category, not an instance. `ORDER_NOT_FOUND` is a code; `ORDER_7841_NOT_FOUND` is not, because the id turns every failure into a fresh string no client can enumerate. The instance belongs in the message or a detail field. When a caller wants "was it a not found", it matches `ORDER_NOT_FOUND`; when it wants "which order", it reads the field. Do not fuse the two.

Publish the code list with the API. A client enumerates the codes in a switch, so if `RATE_LIMITED` and `INVALID_INPUT` are not documented, the client guesses at strings that can change. Treat a new code as an additive contract change: it is safe to ship because no existing client stops matching its current codes.

### The traceId closes the loop

The trace id ties the caller to the server. When a human reports "I hit an error at 10:14", the traceId is the precise identifier that finds the log line. Generate it at the start of the request, put it in both the outgoing envelope and the logging context, and a client that copies it has handed you the exact event to inspect. Without it, "an error happened" and "here is a stack" are two facts you reconcile by hand in the middle of the night. With it, the investigation starts halfway done.

### Shaping the message

The `message` is for the human and belongs in a specific form: a short sentence that names the failure and, where it helps, the offending value. It is the string a person reads, so does not need to be machine-distinct, but it should not contradict the code. When a code says `RATE_LIMITED`, the message should say how soon the caller can retry, not repeat a vague "something went wrong". The message is a legit copy target, so it should read as what an engineer would want the person next to them to understand.

### Centralize: one handler

The mechanical part is that Spring answers every failure from one place. `@RestControllerAdvice` centralizes it.

```java
@RestControllerAdvice
class ApiErrorHandler {
    @ExceptionHandler(OrderConflictException.class)
    ResponseEntity<ErrorDto> handleOrderConflict(OrderConflictException e) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(error("ORDER_ALREADY_PAID", e.getMessage(), e.getTraceId()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ResponseEntity<ErrorDto> handleValidation(MethodArgumentNotValidException e) {
        // collect each field error into details, return 400
        return ResponseEntity.badRequest().body(error("VALIDATION_FAILED", "request failed validation", details(e)));
    }
}
```

The exceptions can be our own domain exceptions, thrown deep in the service, and this single handler maps them all. You throw the exception in the domain and forget about HTTP; the handler knows the status and writes a re-usable envelope. No controller handcrafts its own error body, which is exactly the inconsistency the fix kills.

### The envelope is stable across time

The point of standardization is that the same error always looks the same. A new endpoint still produces the same `404` shape, a validation problem still produces the `400` with the `.details` list. The client's branching is a single place, and the server's mapping is a single file. When you change a message, you change only the text, not the shape, and the client stays.

Diagram: every exception flows to one response shape

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 470" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#33475b"/>
    </marker>
  </defs>

  <rect x="60" y="40" width="300" height="55" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="210" y="63" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Exception thrown</text>
  <text x="210" y="84" text-anchor="middle" font-size="11" fill="#5a6b7a">any layer, any endpoint</text>

  <line x1="210" y1="95" x2="210" y2="142" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="60" y="144" width="300" height="55" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="210" y="167" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Single handler</text>
  <text x="210" y="188" text-anchor="middle" font-size="11" fill="#5a6b7a">@RestControllerAdvice</text>

  <line x1="210" y1="199" x2="210" y2="246" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="60" y="248" width="300" height="55" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="210" y="271" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Standard error envelope</text>
  <text x="210" y="292" text-anchor="middle" font-size="11" fill="#5a6b7a">code  message  details  traceId</text>

  <line x1="210" y1="320" x2="210" y2="367" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="60" y="369" width="340" height="60" rx="10" fill="#eef6ee" stroke="#4a8a4a" stroke-width="1.5"/>
  <text x="210" y="393" text-anchor="middle" font-size="12" font-weight="bold" fill="#2f6b2f">HTTP response</text>
  <text x="210" y="415" text-anchor="middle" font-size="11" fill="#4a8a4a">the same shape for every code</text>
</svg>
```

Everything funnels to the handler, and the handler always emits the same body. The shape does not change with the endpoints; only the code, message, and status inside it do. That consistency is the entire value.

## Real Production Usage

The biggest live APIs all standardize their errors to a machine-readable envelope. Stripe returns JSON with a `type` you can branch on and a `message` for the human, plus the status. GitHub returns an object with a `message` and related details. The reason is the one in this article: a client needs to branch reliably, and a stable field is the only thing it can branch on.

In Spring, the tool is `@RestControllerAdvice` plus `ProblemDetail`, the newer standardized body from Spring 6 that carries a title, a status, a detail, and an instance URI. You are free to design your own envelope instead, but the invariant is the same: one shape, defined once, returned everywhere.

## Common Mistakes

**The `try/catch` in every method.** Catching, logging, and formatting an error body in each controller duplicates the logic and lets it drift. Let exceptions propagate and let the single advice translate them.

**Returning the exception message to the client.** The `500` body that surfaces the internal message leaks implementation detail and reveals the wording. Send a coded error and put the details in the logs.

**Status codes that lie.** A `400` returned for a server bug or a `200` for a failed create, breaks the client's branch point. Convey the true class in the status and the precise reason in the code.

## Interview Perspective

Interviewers probing error handling are checking the contract on the failure path. A weak answer says "return a meaningful HTTP status code." A strong answer holds five dots: a single global handler, one envelope, a stable machine-readable `code` for the client, the human-readable `message`, and a `traceId` that ties the response to the log.

The follow-up that sorts people is "a business rule fails, like an already-paid order. How do you tell the caller?" The strong answer returns `409` with a `code` like `ORDER_ALREADY_PAID`, a message, and a traceId. The weak answer sends a message string and a `400`, leaving the client to parse text. If the candidate cannot name the difference between the status and the application code, they have not shipped a real error contract.

Common follow-ups:

- "What is the difference between the HTTP status and the application error code?"
- "Your endpoint returns an unhelpful `500`. Walk me through how you find out why?"

## Knowledge Check

1. Two different failures both map to HTTP `409`. How does a client tell them apart, and why is that more reliable than comparing `message` strings?
2. A validation failure and a "resource not found" use the same envelope. What is in the envelope that stays the same in both, and what changes?
3. You notice a controller building its own error JSON by hand. What upstream change fixes it, and why does it matter for consistency?

## Key Takeaways

- Return one error envelope for every failure, so the client branches on a stable `code`.
- A single `@RestControllerAdvice` turns every exception into the same response.
- The HTTP status says the class for either; the error code says the precise reason.
- Add a `traceId` so the caller's report and the server's log are the same event.

## What's Next

Errors are now predictable; the next article is about the predictable shape of collections. Pagination, filtering, and sorting is where a list stops being a dump of everything and becomes a bounded, navigable set: the page fields a client walks, a guarantee that the filters apply consistently, and a sort order that behaves the same on every page.

---

This article explains error handling as one standardized envelope, a stable code, a message, and a traceId from a single global handler. It argues that a client branches on the code, not the message, and leaking internals wastes the contract.