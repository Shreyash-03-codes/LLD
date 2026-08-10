# Design an Inventory Management System

## Learning Objectives

- Model the difference between a product and its stock positions, and see why the stock ledger matters more than the product catalog.
- Design stock operations as events with a before and after, so every unit that leaves a warehouse can be explained.
- Handle the concurrency story that is not optional in this system: two orders decrementing the same product's stock at the same moment.

## Introduction

Inventory management is the case study where the data model is the design and everything else is scaffolding. The catalog, the warehouses, the orders: they all exist to serve one question, which is "how much of product X do we have right now, and where?" The trap is that "right now" is a lie in a busy warehouse. Stock is not a number that exists, it is a number that results from a ledger of events, and the systems that treat the number as ground truth are the systems that ship more than they own. Interviewers ask this because it tests whether you can design the boring foundation correctly, and because it has a real concurrency story that the parking lot and the library could only gesture at: every candidate for an e-commerce or logistics backend will live or die by this exact design in their day job.

## Requirements Gathering

Functional requirements:

- The system maintains a product catalog with SKU, name, and metadata.
- Stock is tracked per product, per warehouse, per location within a warehouse.
- Inbound and outbound movements update stock: receiving, selling, adjusting, and returns.
- The system reports current stock levels and flags products that fall below a reorder threshold.
- Every stock change is recorded as a movement event with quantity, direction, and timestamp.

Non-functional requirements:

- Stock updates must be consistent under concurrency; two simultaneous sells of the last unit must not both succeed.
- Stock queries must be cheap and must reflect all committed movements.

Assumptions to state out loud: no multi-currency costing or inventory valuation (FIFO vs weighted average), no batch or expiry tracking, no reservations separate from actual decrements, and stock counts are integer units, not weights or liters. Cut valuation, cut reservations. The interviewer is testing the movement ledger and the concurrency guard, not your accounting knowledge.

## Identifying Core Entities

The entity list is short, and the two classes carrying the weight are the ones beginners leave out.

| Entity | One-line responsibility |
| --- | --- |
| `Product` | The catalog entry: SKU, name, and unit definition. |
| `Warehouse` | A named physical location with a set of storage positions. |
| `StockPosition` | The current count of a product at one warehouse location. |
| `StockMovement` | The ledger event: product, warehouse, quantity delta, reason, timestamp. |
| `StockLedger` | The ordered history of movements that lets the system answer "how did we get here?" |
| `InventoryService` | The facade that applies movements atomically and answers stock queries. |

The distinction that matters is between `StockPosition`, which is the derived current state, and `StockMovement`, which is the recorded truth. Positions can be rebuilt from movements. Movements cannot be rebuilt from positions, and that asymmetry is the whole design.

## Class Design

`Product` is metadata, deliberately thin.

```java
public class Product {
    private final String sku;
    private final String name;

    public Product(String sku, String name) {
        this.sku = sku;
        this.name = name;
    }

    public String getSku() { return sku; }
    public String getName() { return name; }
}
```

`StockPosition` is the current count. Its single interesting method is `apply`, which mutates the count by a signed delta and refuses to go negative. The negative guard is the first line of defense against overselling, and it belongs here, next to the data it protects, not in a service method far away.

```java
public class StockPosition {
    private final String productSku;
    private final String warehouseId;
    private long quantity;

    public StockPosition(String productSku, String warehouseId, long quantity) {
        this.productSku = productSku;
        this.warehouseId = warehouseId;
        this.quantity = quantity;
    }

    public synchronized boolean apply(long delta) {
        if (quantity + delta < 0) {
            return false;
        }
        quantity += delta;
        return true;
    }

    public long getQuantity() { return quantity; }
    public String getProductSku() { return productSku; }
    public String getWarehouseId() { return warehouseId; }
}
```

`StockMovement` is the ledger event. Every reason, inbound or outbound, gets one. The reason field is what makes a ledger explainable: you can ask "why did we go negative on this SKU?" and the movements answer, each one tagged with RECEIVING, SALE, RETURN, or ADJUSTMENT.

```java
public class StockMovement {
    public enum Reason { RECEIVING, SALE, RETURN, ADJUSTMENT }

    private final String productSku;
    private final String warehouseId;
    private final long delta; // positive in, negative out
    private final Reason reason;
    private final Instant occurredAt;

    public StockMovement(String productSku, String warehouseId, long delta, Reason reason) {
        this.productSku = productSku;
        this.warehouseId = warehouseId;
        this.delta = delta;
        this.reason = reason;
        this.occurredAt = Instant.now();
    }

    public String getProductSku() { return productSku; }
    public long getDelta() { return delta; }
    public Reason getReason() { return reason; }
    public Instant getOccurredAt() { return occurredAt; }
}
```

`StockLedger` is the append-only history. In a real system this is a database table; here it is a list. It has one job, record movements, and it does it so the inventory service can reconstruct any position from scratch if it ever needs to.

```java
public class StockLedger {
    private final List<StockMovement> movements = new ArrayList<>();

    public synchronized void record(StockMovement movement) {
        movements.add(movement);
    }

    public List<StockMovement> allMovements(String productSku) {
        return movements.stream()
                .filter(m -> m.getProductSku().equals(productSku))
                .toList();
    }
}
```

`InventoryService` is where the money operation lives: `releaseStock`. This is the atomic apply of a negative delta to a position, paired with a ledger record. The `synchronized` on the position's `apply` is the concurrency guard, and the order matters: apply the delta, and only if it succeeds, record the movement. If you record first and apply second, a failed apply leaves a phantom ledger entry.

```java
public class InventoryService {
    private final Map<String, Product> products = new ConcurrentHashMap<>();
    private final Map<String, StockPosition> positions = new ConcurrentHashMap<>();
    private final StockLedger ledger = new StockLedger();

    private String key(String sku, String warehouseId) {
        return sku + "@" + warehouseId;
    }

    public boolean addStock(String sku, String warehouseId, long qty) {
        StockPosition pos = positions.computeIfAbsent(key(sku, warehouseId),
                k -> new StockPosition(sku, warehouseId, 0));
        if (!pos.apply(qty)) {
            return false;
        }
        ledger.record(new StockMovement(sku, warehouseId, qty, StockMovement.Reason.RECEIVING));
        return true;
    }

    public boolean releaseStock(String sku, String warehouseId, long qty) {
        StockPosition pos = positions.get(key(sku, warehouseId));
        if (pos == null || !pos.apply(-qty)) {
            return false;
        }
        ledger.record(new StockMovement(sku, warehouseId, -qty, StockMovement.Reason.SALE));
        return true;
    }

    public long currentStock(String sku, String warehouseId) {
        StockPosition pos = positions.get(key(sku, warehouseId));
        return pos == null ? 0 : pos.getQuantity();
    }

    public List<StockMovement> history(String sku) {
        return ledger.allMovements(sku);
    }
}
```

Diagram: movements flow into the append-only ledger, the position is derived from it, and the last-unit race resolves inside a single synchronized `apply`.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 920 480" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="ah" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
  </defs>
  <rect width="920" height="480" fill="#ffffff"/>

  <text x="460" y="26" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">Stock is a result, not ground truth</text>

  <text x="30" y="100" font-size="14" font-weight="bold" fill="#1f2937">The ledger is the source of truth</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ah)">
    <line x1="270" y1="134" x2="376" y2="156"/>
    <line x1="270" y1="190" x2="376" y2="175"/>
    <line x1="270" y1="246" x2="376" y2="204"/>
    <line x1="620" y1="175" x2="656" y2="175"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12" fill="#1f2937">
    <rect x="40" y="112" width="230" height="44" rx="7" fill="#dcfce7" stroke="#16a34a"/>
    <text x="155" y="132" text-anchor="middle" font-weight="bold" fill="#14532d">+100</text>
    <text x="180" y="132" font-weight="bold" fill="#14532d">RECEIVING</text>
    <text x="155" y="148" text-anchor="middle" font-size="10.5" fill="#166534">inbound · east WH</text>

    <rect x="40" y="168" width="230" height="44" rx="7" fill="#fee2e2" stroke="#dc2626"/>
    <text x="155" y="188" text-anchor="middle" font-weight="bold" fill="#7f1d1d">-30</text>
    <text x="205" y="188" font-weight="bold" fill="#7f1d1d">SALE</text>
    <text x="155" y="204" text-anchor="middle" font-size="10.5" fill="#991b1b">outbound · east WH</text>

    <rect x="40" y="224" width="230" height="44" rx="7" fill="#fee2e2" stroke="#dc2626"/>
    <text x="155" y="244" text-anchor="middle" font-weight="bold" fill="#7f1d1d">-20</text>
    <text x="205" y="244" font-weight="bold" fill="#7f1d1d">SALE</text>
    <text x="155" y="260" text-anchor="middle" font-size="10.5" fill="#991b1b">outbound · west WH</text>
  </g>

  <rect x="380" y="130" width="240" height="90" rx="8" fill="#f1f5f9" stroke="#94a3b8"/>
  <text x="500" y="156" text-anchor="middle" font-size="13" font-weight="bold" fill="#334155">StockLedger — append-only</text>
  <text x="500" y="180" text-anchor="middle" font-size="15" fill="#475569">+100 · -30 · -20</text>
  <text x="500" y="202" text-anchor="middle" font-size="11" fill="#64748b">events are never deleted</text>

  <rect x="660" y="130" width="220" height="90" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
  <text x="770" y="156" text-anchor="middle" font-size="13" font-weight="bold" fill="#1e3a8a">StockPosition (derived)</text>
  <text x="770" y="184" text-anchor="middle" font-size="22" font-weight="bold" fill="#1d4ed8">quantity = 50</text>
  <text x="770" y="208" text-anchor="middle" font-size="11" fill="#1e40af">keyed SKU@warehouse</text>

  <text x="30" y="300" font-size="14" font-weight="bold" fill="#1f2937">The last-unit race — check and decrement under one lock</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ah)">
    <line x1="300" y1="346" x2="336" y2="346"/>
    <line x1="560" y1="346" x2="596" y2="346"/>
    <line x1="670" y1="372" x2="670" y2="400"/>
    <line x1="670" y1="400" x2="575" y2="410"/>
    <line x1="670" y1="400" x2="670" y2="410"/>
  </g>
  <text x="610" y="396" font-size="11.5" fill="#15803d" font-weight="bold">first</text>
  <text x="700" y="396" text-anchor="middle" font-size="11.5" fill="#b91c1c" font-weight="bold">second</text>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12" fill="#1f2937">
    <rect x="80" y="320" width="220" height="52" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="190" y="343" text-anchor="middle" font-weight="bold" fill="#334155">Order A</text>
    <text x="190" y="359" text-anchor="middle" font-size="11" fill="#64748b">releaseStock(SKU, WH, -1)</text>
    <rect x="340" y="320" width="220" height="52" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="450" y="343" text-anchor="middle" font-weight="bold" fill="#334155">Order B</text>
    <text x="450" y="359" text-anchor="middle" font-size="11" fill="#64748b">releaseStock(SKU, WH, -1)</text>
    <rect x="600" y="320" width="220" height="52" rx="8" fill="#fef3c7" stroke="#f59e0b"/>
    <text x="710" y="343" text-anchor="middle" font-weight="bold" fill="#92400e">synchronized apply(delta)</text>
    <text x="710" y="359" text-anchor="middle" font-size="11" fill="#b45309">one lock serializes both</text>

    <rect x="450" y="410" width="220" height="46" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="560" y="432" text-anchor="middle" font-weight="bold" fill="#14532d">A: 1 - 1 = 0, SALE recorded</text>
    <text x="560" y="448" text-anchor="middle" font-size="10.5" fill="#166534">position never goes negative</text>
    <rect x="660" y="410" width="220" height="46" rx="8" fill="#fee2e2" stroke="#dc2626"/>
    <text x="770" y="432" text-anchor="middle" font-weight="bold" fill="#7f1d1d">B: 0 - 1 &lt; 0, returns false</text>
    <text x="770" y="448" text-anchor="middle" font-size="10.5" fill="#991b1b">no phantom ledger entry</text>
  </g>

</svg>
```

The query side is intentionally boring: read the position map. The history side is where the interview gets interesting, because reconstructing a position from the ledger is the answer to "what if the count is wrong?" and it is the proof that the ledger, not the number, is the source of truth.

## Design Patterns Used

There is no classic GoF pattern at the heart of this system, and the honest answer is to say so. The interesting structure is the append-only ledger, which is an event-sourcing idea in miniature: state is derived from events, events are never deleted. Do not reach for a Decorator to layer overstock checks or a Strategy for allocation policy; the follow-up "where do you enforce the reorder threshold" is answered by a scan of positions and a policy object, not by a pattern. The one pattern-shaped thing worth naming is the Facade in `InventoryService`, and it is a modest one. If the interviewer pushes for a pattern, name the ledger as the real architectural decision and say that reaching for a pattern here would be manufacturing complexity, which is the blunt truth.

## Handling Edge Cases / Concurrency

This is the concurrency case study of the chapter, so go deep. The race is the last unit: two orders for one remaining unit of a SKU arrive together. Both services call `releaseStock`, both read the position, both see quantity 1, both subtract, and the count goes negative. The fix is that `apply` is `synchronized` and the check and the decrement happen under the same lock, so the second caller observes 0 and its `apply` returns false. In a real system with a database, the equivalent is an atomic `UPDATE stock_positions SET quantity = quantity - 1 WHERE sku = ? AND quantity >= 1`, which is the same check-and-decrement made atomic by the database. The candidate who can say "synchronized here, atomic conditional update in the database, same idea" has answered the strongest concurrency question in this chapter.

The second edge is the negative-stock guard. Overselling is a business decision, not a state the system should reach accidentally. The `apply` guard makes negative stock impossible through normal flows, and if the business ever wants backorders, that is a new field and a deliberate choice, not a bug in the guard.

The third edge is the reconstruction proof. If a count looks wrong, the ledger lets you replay all movements for a SKU and see where the drift came from. In a real system that is `SUM(delta) FROM movements WHERE sku = ?` grouped by warehouse, and it is the query that auditors run. Name it.

## Common Mistakes

The most common mistake is the product owning its stock. A `Product` with an `int stock` field and a `decrement()` method means two products in two warehouses cannot exist, because the product has one count, and the interviewer's "how much do we have in the east warehouse" question is unanswerable. Stock is keyed by product and warehouse and position, and the product itself must stay metadata.

The second mistake is mutating the count without a ledger record, or recording without mutating. Either half of the pair on its own produces an inventory that drifts from reality, and the drift is invisible until a stocktake finds it. The movement and the mutation are one transaction, in that order.

The third mistake is a naively non-atomic `releaseStock`. The classic version reads the count, checks it, decrements it, and writes it back in four separate lines, and the last-unit race corrupts it. The candidate who writes that and never mentions the race has built the exact system that oversells at launch.

## Interview Perspective

A weak answer is a catalog and a count. `Product` with stock, `getStock()`, `setStock()`. There is no ledger, no movement, no concurrency story, and the "two orders for the last unit" question is answered with a shrug and a hopeful "we could add a lock." The design has nothing to walk through because nothing can be explained.

A strong answer says "stock is derived from a ledger, positions are the cached result, and the decrement is an atomic check-and-apply." The follow-ups answer themselves. "What if the east warehouse is out and the west warehouse has stock" (the position map is keyed by warehouse, so routing is a lookup, which is exactly why the split exists), "how do you support returns" (a positive delta with a RETURN reason, one ledger entry, no special case), "how do you detect shrinkage" (replay the ledger against a physical count, the difference is the answer). The strongest candidates volunteer the `UPDATE ... WHERE quantity >= 1` phrasing unprompted, because they have run the last-unit race and know it by heart.

## Knowledge Check

1. Two orders for the one remaining unit of a SKU are processed concurrently. Trace both calls through `releaseStock` and explain which order succeeds, which fails, and what the position map and ledger each show afterward.
2. The inventory for a SKU in a warehouse reads 5, but a physical count finds 3. Describe the two things you need to produce to explain the gap, and what question each of them answers.
3. The product catalog and the positions live in separate maps, and the movement carries the SKU rather than the product object. Why does this split make warehouse routing and history reporting possible, and what would break if the product owned its stock count?

## Key Takeaways

- Stock is keyed by product and warehouse and position. The product is metadata, not a count.
- The ledger is the source of truth; the position is the derived cache. Never confuse the two.
- Apply the delta, then record the movement. The order is the correctness.
- The last-unit race is the concurrency story, and the fix is an atomic check-and-decrement, `synchronized` here, conditional update in the database.
- Reconstructing a position from its ledger is the answer to every "why is the count wrong" question, and the query every auditor runs.

## What's Next

Inventory taught you the ledger-and-position split and the atomic decrement. The car rental system reuses the inventory idea, then layers on the thing that changes the arithmetic: time. A car is stock, but a car for this week is a different kind of stock, and calendars replace counts.

---

This article explains how to design an inventory system around the ledger as the source of truth and stock position as the derived count. Its point of view is that stock is a result, not ground truth, and the atomic check-and-apply on release prevents overselling.
