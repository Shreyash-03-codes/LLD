# Design a Payment Processing System

## Learning Objectives

- Design the payment as a state machine with an idempotency key at the front, and understand why this system is where double-charging is a correctness bug, not an edge case.
- Model the gateway as the abstraction between your system and the providers, and know what actually varies behind it.
- Trace the full lifecycle, authorize, capture, refund, webhook, and see where the money can go wrong at each transition.

## Introduction

The payment processing system is the case study where the stakes make every other correctness discussion look like a rehearsal. A lost log line is noise. A double-charged card is a support ticket, a chargeback, and a customer who tells the story for years. The entire design is organized around one property: a payment must move exactly once, no matter how many times the network, the client, the provider, or the operator retries it. The tools are familiar from earlier chapters, the idempotency key from the e-commerce order flow, the state machine from the movie booking, the provider abstraction from the notification channels, but here they are not decorations, they are the product. Interviewers ask this because it tests whether you can hold a state machine and a retry story steady under the pressure of real money.

## Requirements Gathering

Functional requirements:

- A merchant initiates a charge for an amount against a customer's payment method.
- The system routes the payment to a payment provider (card network, wallet, bank transfer) and tracks its status.
- A payment moves through authorize, capture, and settle, and can be refunded, fully or partially.
- The provider reports results asynchronously, via webhook, and the system reconciles those reports with its own state.
- A client can retry a failed or timed-out payment without risk of a double charge.

Non-functional requirements:

- A retried payment must never result in two charges to the same customer for the same intent.
- The system must keep its internal truth even when the provider is slow, down, or ambiguous about what it did.

Assumptions to state out loud: single currency, card and wallet as the only methods, no subscriptions or recurring billing, no escrow or merchant payouts, and authorize-and-capture as one immediate flow rather than a delayed capture. Cut subscriptions and cut payouts. The interviewer wants the transaction lifecycle and the idempotency story, and both are cleaner without a billing engine attached.

## Identifying Core Entities

The entity list is the payment lifecycle with the provider wall in the middle.

| Entity | One-line responsibility |
| --- | --- |
| `Payment` | The transaction record and state machine for one charge intent. |
| `PaymentMethod` | The tokenized card or wallet handle, never raw card data. |
| `PaymentProvider` | The abstraction over a real provider, with charge, capture, refund, and status methods. |
| `IdempotencyRegistry` | The map from client-provided key to the existing payment result. |
| `PaymentService` | The facade where the flow lives: initiate, handle callback, refund. |
| `PaymentEvent` | The webhook payload from a provider. |

The registry deserves its own class because it is doing the deepest work: it is the thing that makes a retry return the original result instead of starting a new charge.

## Class Design

Start with the state machine. `Payment` is the heart, and its states are the lifecycle: PENDING while awaiting the provider, AUTHORIZED when the provider confirmed, CAPTURED when the money moved, FAILED, and REFUNDED. Every transition has a guard, and the guards are what make a double-transition impossible.

```java
public class Payment {
    public enum Status { PENDING, AUTHORIZED, CAPTURED, FAILED, REFUNDED }

    private final String paymentId;
    private final String idempotencyKey;
    private final long amountInCents;
    private final String currency;
    private final PaymentMethod method;
    private Status status = Status.PENDING;

    public Payment(String paymentId, String idempotencyKey, long amountInCents,
                   String currency, PaymentMethod method) {
        this.paymentId = paymentId;
        this.idempotencyKey = idempotencyKey;
        this.amountInCents = amountInCents;
        this.currency = currency;
        this.method = method;
    }

    public boolean markAuthorized() {
        if (status != Status.PENDING) return false;
        status = Status.AUTHORIZED;
        return true;
    }

    public boolean markCaptured() {
        if (status != Status.AUTHORIZED) return false;
        status = Status.CAPTURED;
        return true;
    }

    public boolean markFailed() {
        if (status == Status.CAPTURED) return false; // money moved, no going back
        status = Status.FAILED;
        return true;
    }

    public boolean markRefunded() {
        if (status != Status.CAPTURED) return false;
        status = Status.REFUNDED;
        return true;
    }

    public Status getStatus() { return status; }
    public String getIdempotencyKey() { return idempotencyKey; }
    public String getPaymentId() { return paymentId; }
    public long getAmountInCents() { return amountInCents; }
}
```

The guard discipline is the whole thing. `markFailed` refuses to run after capture, because once the money moved, the failure must become a refund request, not a status flip. That single guard is the difference between a payment system and a payment accident.

`PaymentProvider` is the wall. The interface exists because providers genuinely vary in their APIs, their retry behavior, and their failure shapes, and because the payment service must be able to be tested against a fake provider. The provider returns a result or throws a `ProviderTimeout`, and the distinction between the two is the next class's problem.

```java
public interface PaymentProvider {
    ProviderResult charge(String paymentToken, long amountInCents, String currency);
    ProviderResult refund(String providerChargeId, long amountInCents, String currency);
    ProviderResult getStatus(String providerChargeId);
}
```

`IdempotencyRegistry` is the class that carries the case study. The contract: given the client's idempotency key, return the existing payment if this intent was already seen, otherwise record it and proceed. The atomicity of "check then record" is what stops two concurrent retries from both charging.

```java
public class IdempotencyRegistry {
    private final ConcurrentHashMap<String, String> keyToPayment = new ConcurrentHashMap<>();

    public Optional<String> alreadyProcessed(String idempotencyKey) {
        return Optional.ofNullable(keyToPayment.get(idempotencyKey));
    }

    public boolean claim(String idempotencyKey, String paymentId) {
        return keyToPayment.putIfAbsent(idempotencyKey, paymentId) == null;
    }
}
```

`PaymentService.charge` is the flow, and the ordering is the design. Idempotency first, before anything touches the provider. Then the provider call. Then the state transition. The registry's `claim` and the payment's `markAuthorized` are separate atomic steps, and the whole flow is safe because the registry is checked and claimed before any money-moving call.

```java
public class PaymentService {
    private final IdempotencyRegistry registry;
    private final PaymentProvider provider;
    private final Map<String, Payment> payments = new ConcurrentHashMap<>();

    public Payment charge(String idempotencyKey, long amountInCents,
                          String currency, PaymentMethod method) {
        Optional<String> existing = registry.alreadyProcessed(idempotencyKey);
        if (existing.isPresent()) {
            return payments.get(existing.get()); // retry returns the original result
        }

        Payment payment = new Payment(UUID.randomUUID().toString(), idempotencyKey,
                amountInCents, currency, method);
        payments.put(payment.getPaymentId(), payment);

        if (!registry.claim(idempotencyKey, payment.getPaymentId())) {
            // lost a race against a concurrent retry
            Optional<String> winner = registry.alreadyProcessed(idempotencyKey);
            return payments.get(winner.orElseThrow());
        }

        try {
            ProviderResult result = provider.charge(method.getToken(), amountInCents, currency);
            if (result.isSuccess()) {
                payment.markAuthorized();
                payment.markCaptured();
            } else {
                payment.markFailed();
            }
        } catch (ProviderTimeout e) {
            // ambiguous outcome: leave PENDING, reconcile via webhook or status check
            return payment;
        }
        return payment;
    }

    public void handleWebhook(PaymentEvent event) {
        Payment payment = payments.get(event.getPaymentId());
        if (payment == null) {
            return; // unknown event; log it, do not crash
        }
        switch (event.getType()) {
            case AUTHORIZED -> payment.markAuthorized();
            case CAPTURED -> payment.markCaptured();
            case DECLINED -> payment.markFailed();
        }
    }

    public Payment refund(String paymentId) {
        Payment payment = payments.get(paymentId);
        if (payment.getStatus() != Payment.Status.CAPTURED) {
            throw new IllegalStateException("Only captured payments can be refunded");
        }
        provider.refund(payment.getProviderChargeId(), payment.getAmountInCents(), "USD");
        payment.markRefunded();
        return payment;
    }
}
```

Diagram: the guarded state machine, with the idempotency claim in front of it. Every transition is a guarded method, so an illegal move is impossible by construction.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 880 430" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="ah6" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
  </defs>
  <rect width="880" height="430" fill="#ffffff"/>

  <text x="440" y="28" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">The payment state machine</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ah6)">
    <line x1="240" y1="137" x2="276" y2="137"/>
    <line x1="440" y1="137" x2="466" y2="137"/>
    <line x1="630" y1="137" x2="656" y2="137"/>
    <line x1="360" y1="174" x2="360" y2="226"/>
    <line x1="740" y1="174" x2="740" y2="226"/>
  </g>
  <g stroke="#dc2626" stroke-width="1.8" stroke-dasharray="6 4" fill="none" marker-end="url(#ah6)">
    <polyline points="292,174 292,200 322,200 322,180"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="13" fill="#1f2937">
    <rect x="30" y="100" width="210" height="74" rx="8" fill="#fffbeb" stroke="#f59e0b"/>
    <text x="135" y="122" text-anchor="middle" font-weight="bold" fill="#92400e">IdempotencyRegistry</text>
    <text x="135" y="140" text-anchor="middle" font-size="12" fill="#92400e">claim(key) via putIfAbsent</text>
    <text x="135" y="158" text-anchor="middle" font-size="12" fill="#b45309">atomic — before provider call</text>

    <rect x="280" y="100" width="160" height="74" rx="8" fill="#fef3c7" stroke="#f59e0b"/>
    <text x="360" y="132" text-anchor="middle" font-weight="bold" fill="#92400e">PENDING</text>
    <text x="360" y="152" text-anchor="middle" font-size="12" fill="#b45309">awaiting provider</text>

    <rect x="470" y="100" width="160" height="74" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
    <text x="550" y="132" text-anchor="middle" font-weight="bold" fill="#1e3a8a">AUTHORIZED</text>
    <text x="550" y="152" text-anchor="middle" font-size="12" fill="#1e40af">provider confirmed</text>

    <rect x="660" y="100" width="160" height="74" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="740" y="132" text-anchor="middle" font-weight="bold" fill="#14532d">CAPTURED</text>
    <text x="740" y="152" text-anchor="middle" font-size="12" fill="#15803d">money moved</text>

    <rect x="280" y="230" width="160" height="60" rx="8" fill="#fee2e2" stroke="#dc2626"/>
    <text x="360" y="257" text-anchor="middle" font-weight="bold" fill="#7f1d1d">FAILED</text>
    <text x="360" y="275" text-anchor="middle" font-size="12" fill="#b91c1c">provider declined</text>

    <rect x="660" y="230" width="160" height="60" rx="8" fill="#f3f4f6" stroke="#9ca3af"/>
    <text x="740" y="257" text-anchor="middle" font-weight="bold" fill="#4b5563">REFUNDED</text>
    <text x="740" y="275" text-anchor="middle" font-size="12" fill="#6b7280">money returned</text>
  </g>

  <g font-size="12.5" fill="#475569" text-anchor="middle">
    <text x="310" y="164">retry → original result</text>
    <text x="453" y="128">success</text>
    <text x="643" y="128">capture</text>
    <text x="372" y="205">declined</text>
    <text x="752" y="205">refund</text>
    <text x="300" y="215" fill="#b91c1c">provider timeout →</text>
    <text x="300" y="228" fill="#b91c1c">stays PENDING</text>
  </g>

  <rect x="160" y="348" width="560" height="58" rx="8" fill="#fef2f2" stroke="#fecaca"/>
  <text x="440" y="370" text-anchor="middle" font-size="13" fill="#b91c1c" font-weight="bold">The guard: markFailed() is refused once CAPTURED.</text>

</svg>
```

The `ProviderTimeout` path is the honesty of the whole design. When the provider does not answer, the system does not know whether the money moved, so the payment stays PENDING and the truth arrives later, via a webhook or a status poll. The candidate who refuses to mark a timed-out payment FAILED, because failing it invites the client to retry and double-charge, has the exact instinct the case study tests.

## Design Patterns Used

The pattern here is the Facade in `PaymentService` and the Strategy in `PaymentProvider`, and both are earned: the provider wall is where real variation lives, and the service is the only class the rest of the application sees. The State pattern is present in the payment's guarded transitions, though the enum-switch form is honest for this size. The pattern to name and reject is the Observer for webhooks; a webhook is a plain callback method on the service, not a bus. And the idempotency registry is not a pattern, it is a discipline, which is the strongest thing you can say about it.

## Handling Edge Cases / Concurrency

The concurrency story is the retry, and it has two faces. The concurrent retry: the client's network layer sends the same charge twice at nearly the same moment. Both requests reach the service, both hit the registry, and the `claim` with `putIfAbsent` lets exactly one proceed while the other returns the winner's payment. That is the same compare-and-set you have now seen in five chapters, and here it protects money.

The ambiguous outcome: the provider accepted the charge but the response timed out. The payment is PENDING, the client sees "processing," and the reconciliation path, the webhook or the status poll, resolves the truth. The candidate who knows that PENDING is a real state the system must sit in, rather than a failure to decide, has the senior understanding.

The webhook edges: a webhook can arrive twice, out of order, or for an unknown payment. The guarded state transitions make duplicates harmless, the guards make an out-of-order AUTHORIZED after CAPTURED a no-op, and the unknown event is logged, not thrown. The refund edge: only a captured payment refunds, and a full refund after a partial refund is a separate amount-accounting problem that the single-currency, full-refund scope cut keeps out.

## Common Mistakes

The most common mistake is marking a timed-out payment FAILED. The client retries, the first charge actually went through, and the customer is charged twice. The PENDING state is not indecision, it is the correct posture when the truth is unknown, and the webhook is how the truth arrives.

The second mistake is checking idempotency without atomicity. `if (registry.contains(key)) return existing; registry.put(key, payment);` in two lines, so two concurrent retries both see no existing entry and both charge. The `putIfAbsent` claim is the atomicity, and it is not optional.

The third mistake is a payment class without guarded transitions. A `setStatus(CAPTURED)` called from anywhere means a webhook and a refund path can both mutate the same payment into inconsistent states. The guard methods make illegal transitions impossible by construction, and that is the only safety model a payment system should accept.

## Interview Perspective

A weak answer is a `Payment` with a status field and a `charge()` that calls the provider and sets the status, with no idempotency and no PENDING posture. The interviewer asks "the request times out, what does the client do" and the answer is "retry," and the double-charge is already on the customer's card.

A strong answer says "the idempotency key and the registry are checked and claimed atomically before anything touches the provider, the payment is a state machine with guarded transitions, and a timeout leaves the payment PENDING because the truth is unknown until the webhook arrives." Follow-ups to expect: "what if the webhook never arrives" (a status-poll reconciliation job asks the provider, which is the same ambiguity resolved by a different path), "what if a refund fails" (the refund is itself a retried, idempotent operation against the provider, and the payment stays CAPTURED until it succeeds), "how do you test this" (a fake provider, which is exactly why the provider is an interface). The strongest candidates volunteer the concurrent-retry race and the out-of-order webhook without prompting.

## Knowledge Check

1. A charge request times out at the provider. Walk through the payment's state, what the client observes, and what the two paths are that eventually resolve the payment to a terminal state.
2. The same idempotency key arrives twice concurrently. Trace both requests through the registry and state how many provider calls are made and how many payments exist.
3. A webhook declares AUTHORIZED for a payment that is already CAPTURED. Walk through the transition method and state what happens, and why that behavior is correct rather than an oversight.

## Key Takeaways

- The idempotency key, claimed atomically before any provider call, is the entire double-charge defense.
- The payment is a guarded state machine, and the guards make illegal transitions structurally impossible.
- A timeout leaves PENDING. Deciding when the truth is unknown is the senior move.
- Webhooks are reconciled through the same guarded transitions, and duplicates and out-of-order arrivals are harmless by design.
- The provider is an interface because providers vary and tests need a fake.

## What's Next

The payment system closed the case studies with the discipline that makes retries safe. Chapter 16 turns to the interview itself, and the first skill it interrogates is the one the payment system just proved: the questions you ask before you start designing are the cheapest correctness you will ever buy.

---

This article explains how to design a payment system around the idempotency key claimed atomically before any provider call. Its point of view is that a timed-out payment must stay pending rather than fail, because that is the only way to avoid double charges.
