# Design a Parking Lot

## Learning Objectives

- Model a real-world system whose rules live in the domain, not in a framework: spot fitting, ticketing, and fee calculation.
- See why composition and plain methods beat a tree of `Vehicle` subclasses for this problem.
- Practice the seam placement that lets a follow-up question ("add hourly vs daily pricing") become a small, local change.

## Introduction

The parking lot is the opening act of LLD interviews for a reason. It has enough entities to feel like a system, enough rules to force real decisions, and no tricky concurrency to hide behind. Every software engineer who interviews for a backend role will get this question, or its sibling (the elevator, the vending machine), within their first few loops. Interviewers ask it because it separates people who draw nouns from people who trace verbs. A parking lot does not do anything until a car moves through it, and the whole design hangs off how you model that movement.

## Requirements Gathering

Functional requirements:

- The lot has multiple floors, and each floor has multiple spots of different sizes: compact, large, and accessible.
- A vehicle enters, is assigned a spot it fits in, and is issued a ticket with the entry time.
- A vehicle exits, the fee is computed from the time spent, payment is collected, and the spot is freed.
- Different vehicle types (car, truck) must be checked against spot size.
- The lot should track how many free spots are available per floor.

Non-functional requirements:

- Operations are fast enough that entry and exit gates do not form queues; a lookup and a few field updates per vehicle.
- The design should allow new vehicle types and new pricing schemes without rewriting the core flow.

Assumptions to state out loud: no reservation system, vehicles are not assigned to specific pre-booked spots; the lot is single-entry single-exit so we never handle a car that entered and never left; pricing is time-based only, no lost-ticket or monthly-pass scenarios. An interviewer expects you to cut these. If you try to design reservations, multiple exits, and monthly passes in 45 minutes, you will deliver nothing.

## Identifying Core Entities

The nouns that survive scrutiny are few.

| Entity | One-line responsibility |
| --- | --- |
| `ParkingLot` | Owns floors, issues tickets at entry, settles them at exit. |
| `ParkingFloor` | Owns a set of spots and answers "can you fit this vehicle?" |
| `ParkingSpot` | A single slot with a size and an occupied flag. |
| `Vehicle` | What enters; carries a type that determines which spot fits. |
| `Ticket` | The contract between entry and exit; holds the spot and the entry time. |

Notice what is not on this list. There is no `User`, no `Gate`, no `Payment`. The gate is an implementation detail of how entry and exit happen, not a core entity; the fee logic can live on the ticket or in a small helper. Keep the noun list short and every noun load-bearing.

## Class Design

The design centers on one flow: enter and exit. Everything else hangs off it.

`Vehicle` is the entry point of the whole story, so it is the one place a small hierarchy earns its keep. But keep it to an enum that drives fitting logic, not a class hierarchy with ten subclasses. The interviewer's next question is always "how does a truck park?" and the cleanest answer is "the spot-finding logic checks size compatibility."

```java
public enum VehicleType {
    COMPACT(1),
    LARGE(2),
    TRUCK(3);

    private final int size;
    VehicleType(int size) { this.size = size; }
    public int getSize() { return size; }
}

public class Vehicle {
    private final String licensePlate;
    private final VehicleType type;

    public Vehicle(String licensePlate, VehicleType type) {
        this.licensePlate = licensePlate;
        this.type = type;
    }

    public VehicleType getType() { return type; }
    public int getRequiredSize() { return type.getSize(); }
}
```

`ParkingSpot` is a dumb data holder with a single rule: it can take a vehicle if the vehicle's size fits and the spot is free.

```java
public class ParkingSpot {
    private final String id;
    private final int size;
    private boolean occupied;

    public ParkingSpot(String id, int size) {
        this.id = id;
        this.size = size;
    }

    public boolean canFit(Vehicle vehicle) {
        return !occupied && size >= vehicle.getRequiredSize();
    }

    public void assign() { occupied = true; }
    public void free() { occupied = false; }
    public String getId() { return id; }
}
```

`ParkingFloor` answers the allocation question. The naive implementation loops over spots, which is fine at this scale. If the interviewer pushes on scale, that is the moment to mention a `TreeMap<Integer, List<ParkingSpot>>` keyed by size so a truck only ever scans the spots that can hold it. Do not build that map preemptively; say the loop is fine and show you know where the optimization goes.

```java
public class ParkingFloor {
    private final String id;
    private final List<ParkingSpot> spots;

    public ParkingFloor(String id, List<ParkingSpot> spots) {
        this.id = id;
        this.spots = spots;
    }

    public Optional<ParkingSpot> findAvailableSpot(Vehicle vehicle) {
        return spots.stream()
                .filter(s -> s.canFit(vehicle))
                .findFirst();
    }

    public boolean hasFreeSpot() {
        return spots.stream().anyMatch(s -> !s.isOccupied());
    }
}
```

`Ticket` carries the entry time and the spot. The fee computation belongs here, as a method, because the ticket already has everything the fee needs: the entry time and, implicitly, the current time. A `ParkingFeeService` with a strategy interface is defensible but probably premature; the method version is honest.

```java
public class Ticket {
    private final String id;
    private final ParkingSpot spot;
    private final LocalDateTime entryTime;
    private boolean paid;

    public Ticket(String id, ParkingSpot spot, LocalDateTime entryTime) {
        this.id = id;
        this.spot = spot;
        this.entryTime = entryTime;
    }

    public long feeUntil(LocalDateTime exitTime, long ratePerHour) {
        long minutes = Duration.between(entryTime, exitTime).toMinutes();
        long hours = Math.max(1, (long) Math.ceil(minutes / 60.0));
        return hours * ratePerHour;
    }

    public void markPaid() { paid = true; }
    public ParkingSpot getSpot() { return spot; }
}
```

`ParkingLot` is the facade every external actor talks to. Entry assigns the vehicle a spot, issues a ticket, and updates floor counts. Exit computes the fee, frees the spot, and settles the ticket.

```java
public class ParkingLot {
    private final List<ParkingFloor> floors;
    private final Map<String, Ticket> activeTickets = new HashMap<>();

    public ParkingLot(List<ParkingFloor> floors) {
        this.floors = floors;
    }

    public Optional<Ticket> enter(Vehicle vehicle) {
        for (ParkingFloor floor : floors) {
            Optional<ParkingSpot> spot = floor.findAvailableSpot(vehicle);
            if (spot.isPresent()) {
                ParkingSpot assigned = spot.get();
                assigned.assign();
                Ticket ticket = new Ticket(UUID.randomUUID().toString(), assigned, LocalDateTime.now());
                activeTickets.put(ticket.getId(), ticket);
                return Optional.of(ticket);
            }
        }
        return Optional.empty();
    }

    public long exit(Ticket ticket, long ratePerHour) {
        ticket.getSpot().free();
        activeTickets.remove(ticket.getId());
        return ticket.feeUntil(LocalDateTime.now(), ratePerHour);
    }

    public boolean hasFreeSpot() {
        return floors.stream().anyMatch(ParkingFloor::hasFreeSpot);
    }
}
```

Diagram: the two flows the whole design hangs off, plus the single fitting rule that decides whether a vehicle can use a spot.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 920 450" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="ah5" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
  </defs>
  <rect width="920" height="450" fill="#ffffff"/>

  <text x="460" y="26" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">Two flows, one fitting rule</text>

  <text x="30" y="78" font-size="14" font-weight="bold" fill="#1e3a8a">ENTRY</text>
  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ah5)">
    <line x1="190" y1="127" x2="206" y2="127"/>
    <line x1="380" y1="127" x2="396" y2="127"/>
    <line x1="570" y1="127" x2="586" y2="127"/>
    <line x1="760" y1="127" x2="775" y2="127"/>
  </g>
  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="13" fill="#1f2937">
    <rect x="20" y="95" width="170" height="64" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="105" y="121" text-anchor="middle" font-weight="bold" fill="#334155">Vehicle</text>
    <text x="105" y="139" text-anchor="middle" font-size="12" fill="#64748b">type: TRUCK (size 3)</text>
    <rect x="210" y="95" width="170" height="64" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="295" y="121" text-anchor="middle" font-weight="bold" fill="#334155">ParkingLot.enter()</text>
    <text x="295" y="139" text-anchor="middle" font-size="12" fill="#64748b">for each floor</text>
    <rect x="400" y="95" width="170" height="64" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="485" y="121" text-anchor="middle" font-weight="bold" fill="#334155">findAvailableSpot()</text>
    <text x="485" y="139" text-anchor="middle" font-size="12" fill="#64748b">stream + canFit</text>
    <rect x="590" y="95" width="170" height="64" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="675" y="121" text-anchor="middle" font-weight="bold" fill="#14532d">canFit?</text>
    <text x="675" y="139" text-anchor="middle" font-size="12" fill="#15803d">size ≥ required &amp;&amp; !occupied</text>
  </g>
  <text x="845" y="121" font-size="12.5" fill="#1e40af" font-weight="bold">assign +</text>
  <text x="845" y="137" font-size="12.5" fill="#1e40af" font-weight="bold">Ticket</text>

  <text x="30" y="228" font-size="14" font-weight="bold" fill="#7f1d1d">EXIT</text>
  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ah5)">
    <line x1="190" y1="277" x2="203" y2="277"/>
    <line x1="450" y1="277" x2="463" y2="277"/>
    <line x1="635" y1="277" x2="648" y2="277"/>
  </g>
  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="13" fill="#1f2937">
    <rect x="20" y="245" width="170" height="64" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="105" y="271" text-anchor="middle" font-weight="bold" fill="#334155">scan Ticket</text>
    <text x="105" y="289" text-anchor="middle" font-size="12" fill="#64748b">activeTickets lookup</text>
    <rect x="205" y="245" width="245" height="64" rx="8" fill="#fffbeb" stroke="#f59e0b"/>
    <text x="327" y="271" text-anchor="middle" font-weight="bold" fill="#92400e">Ticket.feeUntil(rate)</text>
    <text x="327" y="289" text-anchor="middle" font-size="12" fill="#b45309">the pricing seam — strategy goes here</text>
    <rect x="465" y="245" width="170" height="64" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="550" y="271" text-anchor="middle" font-weight="bold" fill="#334155">spot.free()</text>
    <text x="550" y="289" text-anchor="middle" font-size="12" fill="#64748b">occupied = false</text>
    <rect x="650" y="245" width="170" height="64" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="735" y="271" text-anchor="middle" font-weight="bold" fill="#14532d">settle</text>
    <text x="735" y="289" text-anchor="middle" font-size="12" fill="#15803d">remove from activeTickets</text>
  </g>

  <text x="460" y="365" text-anchor="middle" font-size="14" font-weight="bold" fill="#1f2937">The one rule that decides everything</text>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12.5">
    <rect x="20" y="382" width="280" height="50" rx="8" fill="#f0fdf4" stroke="#bbf7d0"/>
    <text x="160" y="406" text-anchor="middle" fill="#374151">Car (size 2) → LARGE spot (2)</text>
    <text x="160" y="424" text-anchor="middle" font-weight="bold" fill="#15803d">FITS ✓</text>
    <rect x="320" y="382" width="280" height="50" rx="8" fill="#fef2f2" stroke="#fecaca"/>
    <text x="460" y="406" text-anchor="middle" fill="#374151">Truck (size 3) → COMPACT spot (1)</text>
    <text x="460" y="424" text-anchor="middle" font-weight="bold" fill="#b91c1c">NO ✗</text>
    <rect x="620" y="382" width="280" height="50" rx="8" fill="#f0fdf4" stroke="#bbf7d0"/>
    <text x="760" y="406" text-anchor="middle" fill="#374151">Truck (size 3) → TRUCK spot (3)</text>
    <text x="760" y="424" text-anchor="middle" font-weight="bold" fill="#15803d">FITS ✓</text>
  </g>

</svg>
```

This is a complete system in about 120 lines. That is the target. A parking lot design that needs 400 lines is a design that lost the plot.

## Design Patterns Used

The one pattern that genuinely fits is the Strategy pattern, placed at the pricing seam. The question to ask is not "which patterns can I name?" but "where will the interviewer push?" On a parking lot, the push is almost always pricing. If you have extracted an interface at that seam, the follow-up "add a weekend rate" is a new implementation and a wiring change. So the honest answer here is: one strategy seam at pricing, and no other patterns. No Observer, no Factory, no State machine. The State pattern for "occupied vs free" spots is overkill; a boolean plus two methods does the same job in a quarter of the code, and the interviewer will not be impressed by a pattern you cannot justify when they ask why it is better than the boolean.

## Handling Edge Cases / Concurrency

A basic parking lot has almost no interesting concurrency, and the honest answer is to say so. The genuinely sharp edge is the exit path: what if the same ticket is scanned twice, or a spot is freed twice? The `paid` flag and the `activeTickets` map guard against the double-settle. In a real deployment with multiple entry gates you would need locking on spot assignment so two cars are not handed the same spot, and that is the point where you would mention `synchronized`, an `AtomicBoolean` per spot, or a database row lock in a real system. In the interview, name the race ("two concurrent `enter` calls could pick the same free spot") and the fix ("synchronize the find-and-assign so the check and the update are atomic"), and you have shown more depth than most candidates ever reach on this problem.

## Common Mistakes

The classic failure is the `Vehicle` class hierarchy. `Truck extends Vehicle`, `Bus extends Vehicle`, `ElectricVehicle extends Vehicle`, and suddenly a parking lot has an inheritance tree that is doing no work. Size compatibility is a number; an enum carries it. Every subclass you add forces a decision somewhere, and none of those decisions exist in the requirements.

The second mistake is putting the fee logic in `ParkingLot`. When the interviewer asks "add a weekly flat rate," the candidate discovers the lot now has a pricing rule smeared across the class. The ticket's fee method, or a small strategy behind it, keeps the pricing rule local to the thing that knows the times.

The third mistake is ignoring the exit path. Candidates design a gorgeous entry flow and then a five-line exit. The exit is where the money is, literally. It computes the fee, it frees the spot, it settles the ticket, it handles the missing-ticket case. Shortchanging it reads as not having finished the job.

## Interview Perspective

A weak answer draws `ParkingLot`, then `ParkingLotFloor`, then `ParkingLotFloorRow`, then `ParkingSpot` with four subtypes, then a visitor for payments, and cannot park a car. The classes are fine nouns, but the verbs are missing and there is no walkthrough.

A strong answer says "here is how a car enters and here is how it leaves," and the classes visibly support both. When the interviewer says "what if two cars enter at the same time," the strong candidate points at the find-and-assign loop and names the race without being told. When the interviewer says "add weekend pricing," the strong candidate points at the one seam. Follow-up twists are standard: multi-level (already handled by `floors`), reserved spots (add a `Reservable` flag and check it in `canFit`), different rates per vehicle type (pass a per-type rate into the fee computation or map it in the pricing strategy). Each twist should land as a small, local change.

## Knowledge Check

1. A truck enters a lot where every `LARGE` spot is full but plenty of `COMPACT` spots are free. Trace exactly which methods run and what each returns.
2. Two cars approach two entry gates at the same moment and the only remaining large spot is the same one. Where is the race, and what is the minimal fix that keeps find-and-assign atomic?
3. The business adds a rule: weekdays are hourly, weekends are a flat daily rate. Where does this rule live in the design given, and why is that location better than putting it in `ParkingLot.exit`?

## Key Takeaways

- Keep the entity list short: lot, floor, spot, vehicle, ticket. Every noun must be load-bearing.
- Model vehicle size as an enum, not an inheritance tree.
- Give `Ticket` the fee method; the ticket is the only object that knows the entry time.
- One strategy seam at pricing, nothing else. Most pattern-chasing on this problem is wasted motion.
- Walk both flows end to end. Entry is half the system; exit is the other half, and it is the half with the money.

## What's Next

The parking lot taught you the shape of a classic system: a small set of entities, one seam, two flows. The elevator throws away the "one seam" comfort and forces you to think about a controller that makes decisions, which changes everything about how you split responsibility.

---

This article explains how to design a parking lot with a short entity list and two fully traced flows, entry and exit. Its point of view is that the deliverable is that minimal shape, with vehicle size as an enum and a single pricing seam.
