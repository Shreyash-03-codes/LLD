# Design an E-Commerce Shopping Cart and Order Flow

## Learning Objectives

- Learn the distinction that defines e-commerce: the cart is a session artifact, the order is a transaction, and conflating them is how systems oversell.
- Design the order placement as the single moment where stock is checked and committed, not the cart, where items can sit for days.
- Model the order as a state machine and see where idempotency, the guard against double-submission, belongs in the flow.

## Introduction

The shopping cart looks like a trivial case study: add things to a list, check out, done. The reality is that the cart is the single most mis-modeled object in e-commerce design, and the mistake is always the same one. Candidates put stock on the cart. The cart becomes the thing that knows how many units of an item are left, and the order becomes a copy of the cart. That design works until it does not: a cart can sit for a week while stock drains, and the moment of truth, whether the order can actually be filled, happens at checkout, not at add-to-cart. The cart is a wish list with prices frozen in time. The order is a transaction against current reality. The interview is whether you can keep those two apart, and place the one real concurrency checkpoint exactly where it belongs.

## Requirements Gathering

Functional requirements:

- A user adds items to a cart, changes quantities, and removes items, with the cart persisting across sessions.
- The cart shows line totals and a grand total based on the prices at the time of the calculation.
- Placing an order checks current stock for every line, reserves the units, records the total, and transitions through a defined order lifecycle.
- A user can see their past orders and each order's status.
- A cart item that becomes unavailable is flagged at checkout, not silently dropped.

Non-functional requirements:

- Order placement must be atomic: either every line is filled or none is, and the stock decrement and the order record happen together.
- Double-submitting a checkout must not create two orders.

Assumptions to state out loud: no promotions or coupon codes, no gift wrapping or shipping-cost modeling beyond a flat rate, no multiple currencies, and cart items hold a snapshot of the unit price. Cut promotions, cut shipping zones. The interviewer wants the cart-versus-order split and the checkout transaction, and both are clean only without a pricing engine on top.

## Identifying Core Entities

The entity list is where the case study is won, because the split the whole design depends on is visible in the list.

| Entity | One-line responsibility |
| --- | --- |
| `Cart` | A session-scoped aggregation of `CartItem`s, with quantities and computed totals. |
| `CartItem` | A product, a quantity, and a frozen unit price snapshot. |
| `Product` | The catalog entry with current stock and current price. |
| `Order` | The transaction record: user, lines, total, status, and timestamps. |
| `OrderLine` | A product, a quantity, and the price that was actually charged. |
| `InventoryService` | The stock authority that decrements units atomically. |
| `OrderService` | The checkout flow: validate, reserve, charge, confirm. |

Notice the two services: `InventoryService` owns stock, `OrderService` owns the flow. The cart owns neither. A `Cart` with an `InventoryService` reference is a cart that has grown a responsibility it should not have.

## Class Design

`CartItem` holds a price snapshot. The snapshot is the design decision in miniature: the price shown in the cart is the price at the time the item was added or last viewed, and the order charges a separately captured price, so both can differ from the catalog price without either being "wrong."

```java
public class CartItem {
    private final String productId;
    private int quantity;
    private final long unitPriceInCents; // snapshot

    public CartItem(String productId, int quantity, long unitPriceInCents) {
        this.productId = productId;
        this.quantity = quantity;
        this.unitPriceInCents = unitPriceInCents;
    }

    public void setQuantity(int quantity) { this.quantity = quantity; }
    public long getLineTotalInCents() { return unitPriceInCents * quantity; }
    public String getProductId() { return productId; }
    public int getQuantity() { return quantity; }
    public long getUnitPriceInCents() { return unitPriceInCents; }
}
```

`Cart` is a plain aggregation with a total, deliberately boring. The absence of stock logic in here is the point.

```java
public class Cart {
    private final String cartId;
    private final Map<String, CartItem> items = new LinkedHashMap<>();

    public Cart(String cartId) { this.cartId = cartId; }

    public void addItem(String productId, int quantity, long unitPriceInCents) {
        CartItem existing = items.get(productId);
        if (existing != null) {
            existing.setQuantity(existing.getQuantity() + quantity);
        } else {
            items.put(productId, new CartItem(productId, quantity, unitPriceInCents));
        }
    }

    public void removeItem(String productId) { items.remove(productId); }

    public List<CartItem> getItems() { return new ArrayList<>(items.values()); }

    public long getTotalInCents() {
        return items.values().stream().mapToLong(CartItem::getLineTotalInCents).sum();
    }
}
```

`Product` carries current price and current stock, owned by the catalog side of the system. It is the thing the checkout checks against, and the thing the cart deliberately does not touch.

```java
public class Product {
    private final String productId;
    private final String name;
    private final long currentPriceInCents;
    private final long currentStock;

    public Product(String productId, String name, long currentPriceInCents, long currentStock) {
        this.productId = productId;
        this.name = name;
        this.currentPriceInCents = currentPriceInCents;
        this.currentStock = currentStock;
    }

    public String getProductId() { return productId; }
    public long getCurrentPriceInCents() { return currentPriceInCents; }
    public long getCurrentStock() { return currentStock; }
}
```

`InventoryService` is the stock authority, and its `reserve` is the atomic check-and-apply from the inventory chapter, wearing a reservation hat. The reservation is a separate concept from a hard decrement: it holds stock for the brief payment window, then either converts to a decrement on success or returns on failure. If you have read the movie booking chapter, this is the seat hold again, renamed.

```java
public class InventoryService {
    private final Map<String, Product> products = new ConcurrentHashMap<>();
    private final Map<String, Long> stock = new ConcurrentHashMap<>();

    public synchronized boolean reserve(Map<String, Integer> productQuantities) {
        for (Map.Entry<String, Integer> e : productQuantities.entrySet()) {
            long available = stock.getOrDefault(e.getKey(), 0L);
            if (available < e.getValue()) {
                return false;
            }
        }
        for (Map.Entry<String, Integer> e : productQuantities.entrySet()) {
            stock.merge(e.getKey(), -(long) e.getValue(), Long::sum);
        }
        return true;
    }
}
```

`Order` is the state machine and the transaction record. The transitions are the guard rails: an order is PENDING at creation, CONFIRMED after payment, and the idempotency key lives alongside it.

```java
public class Order {
    public enum Status { PENDING, CONFIRMED, FAILED }

    private final String orderId;
    private final String userId;
    private final String idempotencyKey;
    private final List<OrderLine> lines;
    private final long totalInCents;
    private Status status;

    public Order(String orderId, String userId, String idempotencyKey,
                 List<OrderLine> lines, long totalInCents) {
        this.orderId = orderId;
        this.userId = userId;
        this.idempotencyKey = idempotencyKey;
        this.lines = lines;
        this.totalInCents = totalInCents;
        this.status = Status.PENDING;
    }

    public boolean confirm() {
        if (status != Status.PENDING) return false;
        status = Status.CONFIRMED;
        return true;
    }

    public boolean fail() {
        if (status != Status.PENDING) return false;
        status = Status.FAILED;
        return true;
    }
}
```

`OrderService` is the checkout flow, and the order of operations is the entire design: build the order, reserve stock, then charge. The reserve is what makes the flow atomic. Two checkouts of the last unit, only one reserve succeeds, and the loser's order is marked FAILED and its stock was never touched.

```java
public class OrderService {
    private final InventoryService inventory;
    private final Map<String, Order> orders = new ConcurrentHashMap<>();
    private final Map<String, String> idempotencyStore = new ConcurrentHashMap<>();

    public Order placeOrder(Cart cart, String userId, String idempotencyKey) {
        if (idempotencyStore.containsKey(idempotencyKey)) {
            return orders.get(idempotencyStore.get(idempotencyKey));
        }

        Map<String, Integer> quantities = new HashMap<>();
        List<OrderLine> lines = new ArrayList<>();
        for (CartItem item : cart.getItems()) {
            quantities.put(item.getProductId(), item.getQuantity());
            lines.add(new OrderLine(item.getProductId(), item.getQuantity(),
                    item.getUnitPriceInCents()));
        }

        Order order = new Order(UUID.randomUUID().toString(), userId, idempotencyKey,
                lines, cart.getTotalInCents());

        if (!inventory.reserve(quantities)) {
            order.fail();
            orders.put(order.getOrderId(), order);
            return order;
        }

        order.confirm();
        orders.put(order.getOrderId(), order);
        idempotencyStore.put(idempotencyKey, order.getOrderId());
        return order;
    }
}
```

Diagram: the cart-versus-order split, and the checkout flow where stock is committed exactly once — at reserve time, never at add-to-cart.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 920 440" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="ahd" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
  </defs>
  <rect width="920" height="440" fill="#ffffff"/>

  <text x="460" y="26" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">The cart is a wish list; the order is a transaction</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ahd)">
    <line x1="340" y1="129" x2="576" y2="129"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12.5" fill="#1f2937">
    <rect x="40" y="92" width="300" height="74" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
    <text x="190" y="116" text-anchor="middle" font-weight="bold" fill="#1e3a8a">Cart — session artifact</text>
    <text x="190" y="138" text-anchor="middle" font-size="12" fill="#1e40af">lines with frozen price snapshots</text>
    <text x="190" y="156" text-anchor="middle" font-size="12" fill="#1e40af">no stock, no inventory reference</text>

    <rect x="580" y="92" width="300" height="74" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="730" y="116" text-anchor="middle" font-weight="bold" fill="#14532d">Order — transaction</text>
    <text x="730" y="138" text-anchor="middle" font-size="12" fill="#15803d">own lines, own prices, idempotency</text>
    <text x="730" y="156" text-anchor="middle" font-size="12" fill="#15803d">PENDING → CONFIRMED / FAILED</text>
  </g>
  <text x="458" y="122" text-anchor="middle" font-size="12" fill="#475569">checkout</text>

  <text x="30" y="215" font-size="14" font-weight="bold" fill="#1f2937">The checkout flow — stock is committed once</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ahd)">
    <line x1="200" y1="268" x2="236" y2="268"/>
    <line x1="420" y1="268" x2="456" y2="268"/>
    <line x1="640" y1="268" x2="676" y2="268"/>
    <line x1="770" y1="296" x2="770" y2="326"/>
    <line x1="770" y1="326" x2="550" y2="356"/>
    <line x1="770" y1="326" x2="770" y2="356"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12" fill="#1f2937">
    <rect x="20" y="240" width="180" height="56" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="110" y="263" text-anchor="middle" font-weight="bold" fill="#334155">placeOrder(cart,</text>
    <text x="110" y="278" text-anchor="middle" font-weight="bold" fill="#334155">idempotencyKey)</text>
    <rect x="240" y="240" width="180" height="56" rx="8" fill="#fef2f2" stroke="#dc2626"/>
    <text x="330" y="263" text-anchor="middle" font-weight="bold" fill="#7f1d1d">idempotency check</text>
    <text x="330" y="278" text-anchor="middle" font-size="11" fill="#b91c1c">seen → return existing order</text>
    <rect x="460" y="240" width="180" height="56" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="550" y="263" text-anchor="middle" font-weight="bold" fill="#334155">build Order (PENDING)</text>
    <text x="550" y="278" text-anchor="middle" font-size="11" fill="#64748b">lines from cart snapshots</text>
    <rect x="680" y="240" width="180" height="56" rx="8" fill="#fef3c7" stroke="#f59e0b"/>
    <text x="770" y="263" text-anchor="middle" font-weight="bold" fill="#92400e">reserve(quantities)</text>
    <text x="770" y="278" text-anchor="middle" font-size="11" fill="#b45309">atomic check-and-apply</text>
  </g>
  <text x="330" y="318" text-anchor="middle" font-size="11.5" fill="#b91c1c">double-submission guard — before any mutation</text>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12" fill="#1f2937">
    <rect x="460" y="360" width="180" height="52" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="550" y="383" text-anchor="middle" font-weight="bold" fill="#14532d">reserve OK → confirm</text>
    <text x="550" y="400" text-anchor="middle" font-size="11" fill="#15803d">order CONFIRMED</text>
    <rect x="680" y="360" width="180" height="52" rx="8" fill="#fee2e2" stroke="#dc2626"/>
    <text x="770" y="383" text-anchor="middle" font-weight="bold" fill="#7f1d1d">reserve fails → FAILED</text>
    <text x="770" y="400" text-anchor="middle" font-size="11" fill="#b91c1c">stock untouched</text>
  </g>

</svg>
```

The idempotency check at the top is the double-submission guard: the same checkout retried by a flaky network returns the existing order instead of charging twice. The order of the guard, before the stock check, is what makes retries safe.

## Design Patterns Used

The honest pattern answer here is a modest Facade in `OrderService`, and the real structural idea is the transactional boundary: the checkout is a saga in miniature, reserve then charge, with the reservation as the compensating action if the charge fails. That is worth naming, because it is the pattern real e-commerce actually uses, and it is why the reservation is a separate step from a hard decrement. Do not reach for a Builder for the order (a constructor is fine), do not add a Strategy for shipping (cut from scope), and do not put an Observer on the cart to watch stock. The one place a Strategy could earn its keep is payment, which belongs to its own chapter later in this section.

## Handling Edge Cases / Concurrency

The concurrency story is the double-submission and the last-unit race, and both have homes in this design. The last-unit race: two users each add the last unit to their carts, both place orders, and `reserve` is the single choke point. The first `reserve` decrements, the second sees zero and returns false, and the second order is FAILED with its stock untouched. That is the same atomic check-and-apply as the movie seat, and the walkthrough is identical.

The double-submission race: a user double-clicks checkout or a network retry sends the same request twice. Without the idempotency key, both requests would reserve twice and charge twice. With it, the second request returns the existing order. The key is generated by the client or a gateway at the first request and reused for retries, and the store's `putIfAbsent` behavior is what makes concurrent duplicates collapse to one order.

The edge beyond the races: a cart item whose stock dropped below the cart quantity since the item was added. The checkout fails the reserve and the user sees a flagged line, which is the requirement that unavailable items surface at checkout rather than silently vanishing. And the cart total versus order total: the cart shows snapshot prices, the order charges snapshot prices too, and if the catalog price changed in between, neither is wrong, because the cart's snapshot was the quote the user saw. State that, and the "which price is correct" follow-up answers itself.

## Common Mistakes

The most common mistake is stock on the cart. `CartItem.isAvailable` or a cart-level stock check, so the cart decides whether the checkout can proceed. That design either checks stock at add time, which goes stale by checkout, or checks at checkout, which means the cart has grown an inventory service reference and the responsibility split is already gone. Stock lives in one place, `InventoryService`, and the checkout asks it, once.

The second mistake is a cart that becomes the order. The candidate models one object, the cart, and adds a status field to it. The order is not the cart with a flag, because the order is the transaction: it has its own lines, its own prices, its own idempotency, and its own lifecycle. Merging them means a user who edits their cart after placing an order has edited the order.

The third mistake is no idempotency. A double-click checkout, which happens on every single bad-network checkout in production, creates two orders and two charges. The candidate who says "we just check if the button was clicked twice" has an answer for the UI, not for the network. The idempotency key is the only honest guard.

## Interview Perspective

A weak answer is `Cart` with a `checkout()` method that decrements product stock directly. The interviewer asks "what if the user's cart is a week old" and the answer is silent, because the cart had no snapshot and the stock check, if it exists, ran at add time. The order is a copy of the cart and there is no transaction boundary anywhere.

A strong answer says "the cart is a session artifact with price snapshots, the order is a transaction, and checkout is reserve, then charge, with an idempotency key at the front." Follow-ups to expect: "what if the price changed between cart and order" (the cart showed a snapshot, the order charges the snapshot, and a price-change banner is a UI concern, which is the honest scope line), "what if stock is reserved but the payment fails" (the reservation is released or converted, which is where the saga's compensating step lives, and it belongs to the payment chapter), "what if the user removes an item after checkout" (the order lines are immutable, the cart is a different object, which is exactly why they are separate). The strongest candidates volunteer the last-unit race and the idempotency key unprompted, because they have seen both in production.

## Knowledge Check

1. A user adds the last unit of an item to their cart, waits three days, and checks out. During those days, no stock arrived. Trace the checkout and state which object decides the outcome, and why the cart's earlier stock check, if it had one, could not have decided it correctly.
2. A flaky network resubmits the same checkout request twice concurrently. Walk through `placeOrder` for both requests and explain what each one returns and how many orders exist afterward.
3. The cart's snapshot price is 500 cents, and the catalog price rose to 700 cents before checkout. Which price does the order charge, and why is that defensible rather than a bug?

## Key Takeaways

- The cart is a session artifact with price snapshots. The order is a transaction. Never merge them.
- Stock lives in one place, and the checkout asks it once, at reserve time.
- Checkout is reserve, then charge, with the reservation as the compensating step. That is the transaction boundary.
- The idempotency key is the guard against double-submission, and it runs before anything else mutates.
- The last-unit race resolves at the reserve, and the loser's order fails cleanly with no stock touched.

## What's Next

The order flow introduced the reserve-then-charge boundary and the idempotency guard. The notification system keeps the async handoff but changes the product: the message is no longer a stock unit, it is a fact, and the design problem is orchestrating channels, templates, and the queue that keeps millions of them from flattening a mail server.

---

This article explains how to design an e-commerce flow around the split between the session-scoped cart and the order as a transaction. Its point of view is that merging them is how systems oversell, and the reserve-then-charge boundary plus idempotency key is the whole checkout.
