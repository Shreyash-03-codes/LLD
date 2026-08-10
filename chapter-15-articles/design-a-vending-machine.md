# Design a Vending Machine

## Learning Objectives

- Model a system whose correctness depends on the order of operations, and see why a state machine is the only honest way to express that.
- Handle money as first-class state: what the machine has collected, what it can refund, and when each transition is legal.
- Learn the two bug classes that dominate this problem, invalid transitions and inventory underflow, and how each is prevented.

## Introduction

The vending machine looks like the easiest question in the chapter. Insert money, pick a product, get a product. That is the shape of it, and it is a trap. The vending machine is the classic system where the sequence of actions is the entire specification. Inserting money after the product is dispensed, dispensing without enough money, refunding twice, vending a product that is out of stock: every one of these is a wrong sequence, and every one of them is a bug a naive class model happily produces. Interviewers ask this to see if you recognize that the problem is temporal. The entity list is boring. The state space is not.

## Requirements Gathering

Functional requirements:

- A customer inserts coins or cash, the machine accumulates the inserted amount, and the display shows the running total.
- The customer selects a product by code; if the balance covers the price and the product is in stock, the machine dispenses it and returns change.
- If the balance is insufficient, the machine reports the shortfall.
- The customer can cancel and get a full refund of everything inserted.
- Out-of-stock products cannot be vended and are reported as such.

Non-functional requirements:

- The machine must never be able to reach an illegal combination of states, such as vending a product that was not paid for.
- Refunds and change must be exactly correct; no rounding drift over a shift of transactions.

Assumptions to state out loud: one machine, one coin slot, no product selection during a dispense cycle, change is always available in the machine so we never need a "no change" failure mode. Also, cash and coins are modeled uniformly as inserted amount; the interviewer is not asking you to build a cash drawer reconciliation system. Cut that. If you do not cut it, you will spend the interview counting quarters.

## Identifying Core Entities

The entities are few, and the interesting one is the machine's own internal state.

| Entity | One-line responsibility |
| --- | --- |
| `VendingMachine` | The facade that holds the state, the balance, and the product inventory, and enforces legal transitions. |
| `Product` | A thing for sale: name, price, and stock count. |
| `VendingState` | An enum describing where the machine is in its transaction lifecycle. |
| `Inventory` | Maps product codes to product slots and answers stock queries. |
| `Coin` | A unit of inserted value. |

The domain is small enough that every class earns its place. The class that most candidates forget is `VendingState`, and it is the class the whole interview hinges on.

## Class Design

`Coin` is trivial but should exist so money has a type instead of floating around as an int that means "something."

```java
public enum Coin {
    NICKEL(5), DIME(10), QUARTER(25), DOLLAR(100);

    private final int cents;
    Coin(int cents) { this.cents = cents; }
    public int getCents() { return cents; }
}
```

`Product` is a name, a price in cents, and a stock counter. The stock counter lives here because the product knows how many of itself are left, and the vending decision is "price fits and stock is positive."

```java
public class Product {
    private final String name;
    private final int priceInCents;
    private int stock;

    public Product(String name, int priceInCents, int stock) {
        this.name = name;
        this.priceInCents = priceInCents;
        this.stock = stock;
    }

    public boolean inStock() { return stock > 0; }
    public void dispenseOne() { stock--; }
    public int getPriceInCents() { return priceInCents; }
    public String getName() { return name; }
}
```

`Inventory` is a thin wrapper over a `Map<String, Product>` keyed by the machine's slot codes. It exists so the machine does not reach into a map directly and so stock lookups are a single responsibility.

```java
public class Inventory {
    private final Map<String, Product> products = new HashMap<>();

    public void add(String code, Product product) { products.put(code, product); }

    public Optional<Product> find(String code) { return Optional.ofNullable(products.get(code)); }
}
```

Now the machine. The state machine lives here, and it should be explicit. Three states are enough: `IDLE` (no money inserted, nothing pending), `HAS_MONEY` (money inserted, awaiting a selection or a refund), and `DISPENSING` (a sale is being finalized, further input is ignored). An explicit enum beats two booleans (`hasMoney`, `isDispensing`) because two booleans can represent the illegal combination "hasMoney is true and isDispensing is true," and this enum makes that combination unrepresentable. That is the entire argument for the state machine in one sentence.

```java
public enum VendingState {
    IDLE, HAS_MONEY, DISPENSING
}
```

```java
public class VendingMachine {
    private final Inventory inventory;
    private VendingState state = VendingState.IDLE;
    private int balanceInCents = 0;

    public VendingMachine(Inventory inventory) {
        this.inventory = inventory;
    }

    public int insert(Coin coin) {
        if (state == VendingState.DISPENSING) {
            return balanceInCents; // ignore input during dispense
        }
        balanceInCents += coin.getCents();
        state = VendingState.HAS_MONEY;
        return balanceInCents;
    }

    public int refund() {
        int refund = balanceInCents;
        balanceInCents = 0;
        state = VendingState.IDLE;
        return refund;
    }

    public DispenseResult select(String code) {
        if (state != VendingState.HAS_MONEY) {
            return DispenseResult.failure("Insert money first");
        }
        Optional<Product> productOpt = inventory.find(code);
        if (productOpt.isEmpty() || !productOpt.get().inStock()) {
            return DispenseResult.failure("Product unavailable");
        }
        Product product = productOpt.get();
        if (balanceInCents < product.getPriceInCents()) {
            return DispenseResult.failure(
                "Need " + (product.getPriceInCents() - balanceInCents) + " more cents");
        }

        state = VendingState.DISPENSING;
        product.dispenseOne();
        int change = balanceInCents - product.getPriceInCents();
        balanceInCents = 0;
        state = VendingState.IDLE;
        return DispenseResult.success(product.getName(), change);
    }
}
```

Diagram: the machine as a state machine. Every public method is a guarded transition, and the enum makes the illegal "has money while dispensing" state unrepresentable.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 940 330" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="ahb" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
  </defs>
  <rect width="940" height="330" fill="#ffffff"/>

  <text x="470" y="26" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">The vending machine state machine</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ahb)">
    <line x1="250" y1="160" x2="356" y2="160"/>
    <line x1="570" y1="160" x2="676" y2="160"/>
    <polyline points="440,150 440,108 510,108 510,150"/>
    <polyline points="465,220 465,262 145,262 145,224"/>
    <polyline points="785,220 785,300 145,300 145,224"/>
  </g>
  <g stroke="#dc2626" stroke-width="1.8" stroke-dasharray="6 4" fill="none" marker-end="url(#ahb)">
    <polyline points="735,150 735,108 825,108 825,150"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle">
    <rect x="40" y="150" width="210" height="70" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
    <text x="145" y="185" fill="#1e3a8a">IDLE</text>
    <text x="145" y="204" font-size="12" fill="#1e40af">no money, nothing pending</text>
    <rect x="360" y="150" width="210" height="70" rx="8" fill="#fef3c7" stroke="#f59e0b"/>
    <text x="465" y="185" fill="#92400e">HAS_MONEY</text>
    <text x="465" y="204" font-size="12" fill="#b45309">awaiting selection / refund</text>
    <rect x="680" y="150" width="210" height="70" rx="8" fill="#ede9fe" stroke="#8b5cf6"/>
    <text x="785" y="185" fill="#4c1d95">DISPENSING</text>
    <text x="785" y="204" font-size="12" fill="#6d28d9">finalizing sale, input ignored</text>
  </g>

  <g font-size="12.5" fill="#475569" text-anchor="middle">
    <text x="305" y="148">insert(coin)</text>
    <text x="625" y="148">select() — valid</text>
    <text x="475" y="100">insert(coin) top-up</text>
    <text x="305" y="256">refund() — zero balance</text>
    <text x="465" y="294">dispense + return change</text>
    <text x="780" y="100" fill="#b91c1c">insert ignored</text>
  </g>

</svg>
```

The `DispenseResult` is a tiny value object carrying success or failure plus the change, so the machine can return both outcomes without throwing exceptions for control flow. The critical detail is the order inside `select`: validate state, validate stock, validate balance, then transition. Every one of those guards exists because someone once watched a machine vend a bag of chips it did not have and money it did not receive.

## Design Patterns Used

The State pattern is the pattern here, and unlike the parking lot, it is not optional. The vending machine's methods behave differently based on `state`, and an enum switch is the minimal honest implementation. Do not build a class-per-state hierarchy with `IdleState implements VendingState` unless the interviewer pushes toward it; for three states with simple transitions, that hierarchy is ceremony that doubles your class count without adding a single safety property. The enum approach gives you the same invariants with less code. If the interviewer asks "what happens when you add a 'maintenance' state," that is the moment to show you know how the enum grows: add the value, add the guard clauses.

There is no Observer, no Command, no Strategy needed here. The vending machine's interface is a few synchronous calls; an event bus between the coin slot and the dispenser would be solving a problem the requirements do not contain.

## Handling Edge Cases / Concurrency

This system has no real concurrency if it is a single machine driven by one customer at a time, which is the standard assumption, and say so out loud. What it does have is temporal edge cases, which are the substitute for concurrency in this problem, and they are where the marks are.

The double-refund: `refund()` sets the balance to zero before returning it, so calling it twice refunds zero on the second call. The insert-during-dispense: guarded by the state check. The out-of-stock race between two customers: in a real machine, one customer at a time, so a single-threaded model is correct, but if the interviewer asks about a kiosk serving a queue, the fix is serializing the `select` call, because decrementing stock and reading stock must be atomic. Name that, and you have covered the concurrency angle honestly: this machine is single-user, and if it were not, the single choke point is `select`.

## Common Mistakes

The most common mistake is modeling the machine as a pile of getters. `insert(coin)` returns void, `select(code)` returns the product, and the interview ends. That design works in the happy path and is undefendable on any edge case, because there is no state to reason about. The machine's current balance, its state, and the transition rules are the product being designed. Leaving them out is like designing a bank without a balance field.

The second mistake is using exceptions for the failure cases. `throw new NotEnoughBalanceException()` from `select` is the wrong shape. The out-of-stock and short-balance cases are expected outcomes of a normal interaction, not exceptional conditions. A result type carries both the outcome and the change, and it lets the calling code decide. Interviewers notice this.

The third mistake is forgetting that `insert` should also work to top up after a failed selection. The state transition from `HAS_MONEY` back to `HAS_MONEY` on a second coin is legal, and designs that model "insert only works from IDLE" break the most common real interaction, which is inserting a dollar, failing on a two-dollar item, and adding more. Walk that flow before you call the design done.

## Interview Perspective

A weak answer draws `VendingMachine`, `Product`, `Coin`, and then nothing that enforces order. Ask them to insert a coin, select a product, and then select a second product with a zero balance, and the design happily vends again. The absence of state is the absence of the system.

A strong answer says "the machine is a state machine, IDLE, HAS_MONEY, DISPENSING, and every public method is a transition between legal states." That sentence is worth more than a full class diagram, because it is the thing that makes every edge case tractable. Follow-ups to expect: "what if the customer presses cancel after the product is dispensed" (no refund, the transaction is already closed, the state went to IDLE), "what if a product costs more than what a single coin can cover" (the balance accumulates, that is why `insert` is additive), "what if the machine runs out of change" (the requirement cut said change is always available; if asked to lift that cut, the machine needs a coin-inventory model and a refusal path, and that is where the design grows). Strong candidates volunteer the "no refund after dispense" rule before being asked, because they traced the state transitions and noticed it themselves.

## Knowledge Check

1. A customer inserts a dollar, selects a product that costs two dollars, and the machine correctly refuses. Trace the machine's state and balance through that interaction, then through the customer inserting another dollar and re-selecting.
2. Two booleans, `hasMoney` and `isDispensing`, can represent the state "hasMoney true and isDispensing true," which should never exist. Give a concrete sequence of calls that produces that illegal combination in a boolean-based design, and explain why the enum cannot produce it.
3. The business adds a maintenance mode where the machine can be unlocked, serviced, and relocked, and must refuse all customer actions while open. Where does this change land in the given design?

## Key Takeaways

- The vending machine is a temporal problem. The state enum is the design; the classes are just furniture.
- Guard every transition: state, stock, balance, in that order, before any mutation.
- Return result types for expected failures, not exceptions.
- `refund` zeroes the balance before returning it. Double-calls must return zero.
- Walk the top-up flow (failed selection, add money, re-select) before declaring it done. It is the most common real interaction.

## What's Next

The vending machine taught you to model order of operations as explicit state. The ATM keeps the money theme but moves the money behind a bank, so the machine no longer owns the source of truth, and that single change redraws the whole system.

---

This article explains how to design a vending machine as a state machine, IDLE, HAS_MONEY, and DISPENSING, where every method is a guarded transition. Its point of view is that the state enum is the product, making the worst bug classes structurally impossible.
