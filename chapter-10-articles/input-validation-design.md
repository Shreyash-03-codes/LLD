# Input Validation Design

## Learning Objectives

- Place validation in the layer that owns the rule: the DTO for syntax, the domain object for invariants, the service for cross-object rules.
- Enforce syntax rules with Jakarta Bean Validation annotations, and keep the domain from duplicating them.
- Reject a bad request with a predictable, standardized error so the caller gets one clear answer.

## Introduction

Input validation is the gate on the inside of your system. A malformed email, an empty order, a negative quantity, a number where a string belongs. If you do not stop it at the door, it stops you later, in the domain, in a transaction, in a foreign key that throws at the worst time.

The job is not just to reject. It is to reject in the right layer, so each kind of rule lives once and a request fails the moment it does something that could have been caught earlier.

## Problem Statement

There are three ways validation goes wrong, and you have inherited all of them at least once.

The first is validation by hand in the controller. Every endpoint begins with a wall of `if`: `if (name == null) throw ...`, `if (amount.compareTo(ZERO) < 0) throw ...`. The checks are scattered, duplicated between controllers, and impossible to audit. A new endpoint leaves out the quantity check and a negative value slides past the whole layer.

The second is trusting the caller. The system accepts the request as fact, so `quantity: -1` goes straight into the domain, then into a total, then into a report nobody reads for two weeks. The corruption is the one the missing validation set up.

The third and least obvious is checking the same thing in two places for different reasons. The DTO carries `@Min(0)` and the domain method also checks the bound, so when the business changes the minimum you fix the annotation and forget the domain, or the reverse, and the two drift until a test catches it.

## Core Concept

### Three layers, three jobs

Validation belongs in three homes, and each check belongs in exactly one of them.

The syntax of the payload, whether a field is present, non-null, the right type, in range, belongs to the DTO, expressed as Bean Validation constraints. The DTO represents the wire shape, so the shape rules live there.

The business fact belongs to the domain. A quantity cannot be zero or negative, an order cannot be created empty, an operation can only run in one state. These are the aggregate invariants from the modeling chapter, and they live on the object, expressed in its constructor and its methods.

The cross-object rule belongs to the service. An eligibility check that must see the customer and the order together, a transfer that must see two accounts. When no single object owns the whole rule, a service coordinates and validates the span.

The rule that keeps the split honest: the DTO checks what the shape tells it, the domain checks what it owns, and the service checks only the span. When the DTO and the domain both need "quantity positive", the rule lives once, on the domain, and the DTO does not duplicate it.

### Bean validation on the DTO

The syntax checks land on the DTO as `jakarta.validation` constraints, and the controller triggers them with `@Valid`:

```java
public class CreateOrderRequest {
    @NotBlank
    private String customerId;

    @NotEmpty
    private List<CreateOrderLine> lines;

    @Email
    private String notificationEmail;
}
```

```java
@PostMapping
public OrderDto create(@Valid @RequestBody CreateOrderRequest request) {
    return orderService.create(request);
}
```

For a missing id, an empty list, or a malformed email, the framework answers `400` with the messages derived from those annotations. That is the whole DTO job. It is shape checking at the boundary, and it is not trying to answer "is this a valid business action", which no single annotation can know.

### The domain owns the invariants

The wrong move is dressing domain invariants as DTO annotations. Try to express "the total of the lines cannot be negative" with annotations and you end up with a `@AssertTrue` method or a custom constraint that has to see the whole object anyway. That rule is a domain invariant, and the domain should refuse it.

```java
public record Quantity(long value) {
    public Quantity {
        if (value <= 0) {
            throw new IllegalArgumentException("quantity must be positive");
        }
    }
}
```

Now a `Quantity` with a non-positive value cannot exist, so no path can put a zero quantity into the system. The DTO answers "is the JSON shape valid?", the domain answers "is the business fact valid?", and the two do not duplicate each other. A value the domain accepted is a value that was already valid.

### The service checks the span

The rules that reach across objects live in the service, where every piece is loaded. The checkout needs the customer and the order, and no single domain object owns that whole check. The service calls each object's guard, then runs the cross rule:

```java
public Order checkout(CheckoutRequest request) {
    Customer customer = customers.find(request.customerId());
    customer.requireEligible();
    Order order = Order.create(customerId, basket);
    order.submit();
    orders.save(order);
    return order;
}
```

`requireEligible` belongs to the customer, `submit` belongs to the order, and the service is only the place that sees both. The domain rules are never re-declared here; they are called.

Diagram: the validation gates a request passes

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 470" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#33475b"/>
    </marker>
  </defs>

  <rect x="60" y="40" width="300" height="60" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="210" y="64" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">DTO syntax</text>
  <text x="210" y="86" text-anchor="middle" font-size="11" fill="#5a6b7a">@Valid  @Email  @NotBlank</text>

  <line x1="210" y1="100" x2="210" y2="148" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="360" y1="70" x2="430" y2="70" stroke="#9aa7b4" stroke-width="1.5" stroke-dasharray="5 4" marker-end="url(#arrow)"/>
  <rect x="436" y="45" width="120" height="50" rx="8" fill="#fdecea" stroke="#a94442" stroke-width="1.5"/>
  <text x="496" y="75" text-anchor="middle" font-size="12" fill="#a94442">400 rejected</text>

  <rect x="60" y="150" width="300" height="60" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="210" y="174" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Domain invariant</text>
  <text x="210" y="196" text-anchor="middle" font-size="11" fill="#5a6b7a">object refuses bad fact</text>

  <line x1="210" y1="210" x2="210" y2="258" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="360" y1="180" x2="430" y2="180" stroke="#9aa7b4" stroke-width="1.5" stroke-dasharray="5 4" marker-end="url(#arrow)"/>
  <rect x="436" y="155" width="120" height="50" rx="8" fill="#fdecec" stroke="#a94442" stroke-width="1.5"/>
  <text x="496" y="185" text-anchor="middle" font-size="12" fill="#a94442">400 rejected</text>

  <rect x="60" y="260" width="300" height="60" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="210" y="284" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Service rule</text>
  <text x="210" y="306" text-anchor="middle" font-size="11" fill="#5a6b7a">spans more than one object</text>

  <line x1="210" y1="320" x2="210" y2="368" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="360" y1="290" x2="430" y2="290" stroke="#9aa7b4" stroke-width="1.5" stroke-dasharray="5 4" marker-end="url(#arrow)"/>
  <rect x="436" y="265" width="120" height="50" rx="8" fill="#fdecec" stroke="#a94442" stroke-width="1.5"/>
  <text x="496" y="295" text-anchor="middle" font-size="12" fill="#a94442">400 rejected</text>

  <rect x="60" y="370" width="300" height="60" rx="10" fill="#eef6ee" stroke="#4a8a4a" stroke-width="1.5"/>
  <text x="210" y="394" text-anchor="middle" font-size="12" font-weight="bold" fill="#2f6b2f">Accepted</text>
  <text x="210" y="416" text-anchor="middle" font-size="11" fill="#4a8a4a">service runs the operation</text>
</svg>
```

A request drops through the three gates in order. Each gate rejects what it owns and nothing else, so the DTO does not guess at the business, and the service does not repeat a shape check. A request that clears all three is a request the domain already trusts.

## Real Production Usage

Spring Boot and Bean Validation make the DTO gate pure declaration. The `spring-boot-starter-validation` brings in `jakarta.validation`, and a `@Validated` controller class enables method-level constraint checks. A failed `@Valid` raises `MethodArgumentNotValidException`, which a global handler answers with a `400`, and the standardized error body for that is the next article's job.

The service layer also gets argument validation with `@Validated` on the class and `@Min`, `@NotNull`, on parameters, so the service boundary is checked too. The layered habit that sticks is the discipline: DTO for shape, domain for truth, service for span. A team that pushes every rule into one `@Valid` annotation either double checks the domain or sneaks a business rule into a place it cannot speak.

## Common Mistakes

**Validation in the controller, by hand.** A pile of `if/throw` at the top of each method duplicates the annotations and the domain. Put the shape on the DTO, the truth on the domain, and the span on the service.

**The domain invariant expressed as a DTO constraint.** A `@Min` where the real rule is a business invariant makes the boundary lie about who owns it. The domain can see the aggregate; the DTO cannot.

**Trusting the caller by default.** An API with no gate lets a negative quantity reach the domain, which then refuses it with a generic `IllegalArgumentException` the caller cannot branch on. The `400` at the gate is what the caller should have seen first, in a shape it can read.

## Interview Perspective

Interviewers probing validation are testing where you draw the layering line. A weak answer says "add `@Valid` and the annotations do the work." A strong answer splits it into three: the DTO checks the shape with Bean Validation, the domain owns the invariants, the service runs the cross-object rules, and the controller only carries `@Valid`.

The follow-up that sorts people is "should a business rule like quantity positive be a DTO annotation or a domain check?" The strong answer is the domain, because a single annotation cannot see the aggregate and rules like that belong in the object. The candidate who says "put `@Min` on the DTO" is describing syntax validation, not a business invariant, and missing the difference is the failure point.

Common follow-ups:

- "Your DTO has `@NotBlank` but the service must still handle the missing entity. What went wrong?"
- "Where does a cross-field rule like total cannot exceed the limit live?"

## Knowledge Check

1. A JSON body arrives with `quantity: -1`. What checks it at the boundary, and why does the domain still have to refuse it?
2. The same "positive quantity" rule appears as a `@Min` on the DTO and as a guard in the domain. Why is that a problem, and which one wins?
3. A rule needs the customer and the order in hand before a checkout proceeds. Which validation home owns it, and what do the other two do instead?

## Key Takeaways

- Syntax goes on the DTO, invariants on the domain, cross-object rules on the service.
- `@Valid` triggers Bean Validation, which answers a `400` on the bad shape.
- The domain refuses a bad actor in the constructor, so no path can get around the bound.
- The service calls each object's guard and adds only the span; it does not re-declare the rule.

## What's Next

The gate has rejected a request, and the next article is about what that rejection looks like on the wire. Error handling and standardized error responses covers returning a code and a message that means the same thing every time, a stable envelope a caller can branch on, and the shape that makes sure the client is told why it failed, not just that it did.

---

This article explains input validation as three gates, syntax on the DTO, invariants on the domain, and cross-object rules in the service. It argues the same rule appears once, on the layer that owns it, and a validation is not a domain invariant.