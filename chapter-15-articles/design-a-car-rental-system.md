# Design a Car Rental System

## Learning Objectives

- Model availability over intervals, not instants, and see why a boolean "isAvailable" field is the wrong model for anything that is rented for a duration.
- Design the reservation as the transaction that holds a car for a date range, with overlap as the central integrity rule.
- Build the "list available cars for these dates" query that every customer flow starts with, and know where the concurrency guard goes.

## Introduction

The car rental system is the bridge between the inventory chapter and the booking chapter. A car is inventory, but it is inventory with a calendar. A car is either on the lot or on the road right now, but that instantaneous fact is nearly useless to a customer, who wants to know whether a car is free next Tuesday through Friday. The moment "right now" becomes "these dates," the availability model changes shape: a boolean becomes a set of intervals, and the set of intervals is the design. Interviewers ask this because time-based availability is the skill underneath every rental, booking, and scheduling system, and because it is the cleanest setup for the overlap-check and concurrency questions that follow.

## Requirements Gathering

Functional requirements:

- The system maintains a fleet of vehicles with type (sedan, SUV, truck), model, and daily rate.
- A customer searches for vehicles available in a date range and books one.
- A reservation holds a specific vehicle for a specific pickup and return date.
- The system tracks a reservation through confirmed, ongoing, and completed states, and handles cancellations.
- A vehicle can only be booked once per overlapping period.

Non-functional requirements:

- Availability checks for a date range must be fast, and the overlap check must be correct under concurrent booking.
- A customer's booking of a car must not silently invalidate another customer's existing booking.

Assumptions to state out loud: no hourly rentals or minimum-duration rules, no add-ons (insurance, GPS, child seats), no vehicle condition or mileage tracking, no fleet replenishment, and cancellations are full refunds with no cancellation fee. Cut add-ons and cut mileage; the interviewer wants the reservation model, not a pricing spreadsheet.

## Identifying Core Entities

The entity list is the same shape as a library system with one structural upgrade: the loan became a reservation, and the copy became a vehicle with a calendar of reservations.

| Entity | One-line responsibility |
| --- | --- |
| `Vehicle` | A physical car: license plate, type, model, daily rate. |
| `Customer` | A renter with a license and contact details. |
| `Reservation` | A booking that holds a vehicle for a pickup and return date. |
| `VehicleCalendar` | The set of a vehicle's reservations, answering "are you free on these dates?" |
| `RentalService` | The facade that searches, books, and returns vehicles. |

The entity that beginners miss is `VehicleCalendar`. It is the thing that makes "available for a range" a first-class question instead of a scan through all reservations by hand.

## Class Design

`Vehicle` is a physical asset, metadata plus a rate. It knows nothing about reservations; the calendar knows that.

```java
public class Vehicle {
    private final String licensePlate;
    private final String model;
    private final VehicleType type;
    private final long dailyRateInCents;

    public Vehicle(String licensePlate, String model, VehicleType type, long dailyRateInCents) {
        this.licensePlate = licensePlate;
        this.model = model;
        this.type = type;
        this.dailyRateInCents = dailyRateInCents;
    }

    public String getLicensePlate() { return licensePlate; }
    public VehicleType getType() { return type; }
    public long getDailyRateInCents() { return dailyRateInCents; }
}
```

`Reservation` is the transaction. It carries the dates, the vehicle, and the customer, and it exposes the two interval methods every other part of the system needs: `overlaps` and `contains`. The overlap rule is the integrity rule of the whole system: two reservations overlap if each one's start is before the other's end. Get that one comparison right and the entire calendar works.

```java
public class Reservation {
    private final String id;
    private final Vehicle vehicle;
    private final Customer customer;
    private final LocalDate pickupDate;
    private final LocalDate returnDate;
    private ReservationStatus status = ReservationStatus.CONFIRMED;

    public Reservation(String id, Vehicle vehicle, Customer customer,
                       LocalDate pickupDate, LocalDate returnDate) {
        this.id = id;
        this.vehicle = vehicle;
        this.customer = customer;
        this.pickupDate = pickupDate;
        this.returnDate = returnDate;
    }

    public boolean overlaps(LocalDate start, LocalDate end) {
        return pickupDate.isBefore(end) && start.isBefore(returnDate);
    }

    public void cancel() { status = ReservationStatus.CANCELLED; }
    public void complete() { status = ReservationStatus.COMPLETED; }

    public Vehicle getVehicle() { return vehicle; }
    public LocalDate getPickupDate() { return pickupDate; }
    public LocalDate getReturnDate() { return returnDate; }
    public ReservationStatus getStatus() { return status; }
}
```

`VehicleCalendar` is the availability model. It keeps the active reservations for one vehicle and answers the single question every flow depends on: are you free for this range? The answer is a scan over the vehicle's reservations, which is fine at fleet scale and is the honest thing to say.

```java
public class VehicleCalendar {
    private final Vehicle vehicle;
    private final List<Reservation> reservations = new ArrayList<>();

    public VehicleCalendar(Vehicle vehicle) { this.vehicle = vehicle; }

    public boolean isFree(LocalDate start, LocalDate end) {
        for (Reservation r : reservations) {
            if (r.overlaps(start, end)) {
                return false;
            }
        }
        return true;
    }

    public synchronized boolean tryAdd(Reservation reservation) {
        LocalDate start = reservation.getPickupDate();
        LocalDate end = reservation.getReturnDate();
        if (!isFree(start, end)) {
            return false;
        }
        reservations.add(reservation);
        return true;
    }

    public Vehicle getVehicle() { return vehicle; }
}
```

The `synchronized` on `tryAdd` is the concurrency guard, and it matters for a specific reason: the check-then-add is the race. Without it, two customers can both see the car free for the same weekend and both book it, because both check the calendar and both add before either write is visible. The lock makes check and add one unit. This is the same atomic check-and-apply you saw on the inventory position, wearing a calendar instead of a count.

`RentalService` is the facade. Search iterates the fleet and filters by the calendar; book calls `tryAdd` and only records a reservation if the calendar accepted it. The ordering is the whole trick: the calendar is the gatekeeper, not the service.

```java
public class RentalService {
    private final List<VehicleCalendar> fleet = new ArrayList<>();

    public void registerVehicle(Vehicle vehicle) {
        fleet.add(new VehicleCalendar(vehicle));
    }

    public List<Vehicle> searchAvailable(VehicleType type, LocalDate start, LocalDate end) {
        return fleet.stream()
                .filter(c -> c.getVehicle().getType() == type)
                .filter(c -> c.isFree(start, end))
                .map(VehicleCalendar::getVehicle)
                .toList();
    }

    public Optional<Reservation> book(Vehicle vehicle, Customer customer,
                                      LocalDate start, LocalDate end) {
        VehicleCalendar calendar = fleet.stream()
                .filter(c -> c.getVehicle().equals(vehicle))
                .findFirst()
                .orElseThrow();
        Reservation reservation = new Reservation(UUID.randomUUID().toString(),
                vehicle, customer, start, end);
        if (!calendar.tryAdd(reservation)) {
            return Optional.empty();
        }
        return Optional.of(reservation);
    }
}
```

The walkthrough is short and satisfying: search asks each calendar, book asks the chosen calendar, and the calendar says yes or no, atomically.

Diagram: one vehicle's calendar as intervals, showing an overlapping query (rejected), a free query (accepted), and the boundary rule that lets one reservation end exactly when another starts.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 730 290" font-family="Helvetica, Arial, sans-serif">
  <g stroke="#e5e7eb">
    <line x1="154" y1="40" x2="154" y2="180"/>
    <line x1="218" y1="40" x2="218" y2="180"/>
    <line x1="282" y1="40" x2="282" y2="180"/>
    <line x1="346" y1="40" x2="346" y2="180"/>
    <line x1="410" y1="40" x2="410" y2="180"/>
    <line x1="474" y1="40" x2="474" y2="180"/>
  </g>

  <rect x="154" y="40" width="192" height="34" rx="6" fill="#dc2626"/>
  <text x="10" y="55" font-size="12" fill="#374151" font-weight="bold">Query A</text>
  <text x="10" y="69" font-size="11" fill="#6b7280">Tue–Thu</text>
  <text x="548" y="59" font-size="12" fill="#dc2626" font-weight="bold">overlaps R1 → rejected</text>

  <rect x="282" y="90" width="128" height="34" rx="6" fill="#16a34a"/>
  <text x="10" y="105" font-size="12" fill="#374151" font-weight="bold">Query B</text>
  <text x="10" y="119" font-size="11" fill="#6b7280">Thu–Fri</text>
  <text x="548" y="109" font-size="12" fill="#16a34a" font-weight="bold">fits the gap → accepted</text>

  <rect x="90" y="140" width="448" height="40" rx="8" fill="#f3f4f6" stroke="#d1d5db"/>
  <rect x="90" y="140" width="192" height="40" rx="6" fill="#2563eb"/>
  <rect x="410" y="140" width="128" height="40" rx="6" fill="#2563eb"/>
  <rect x="154" y="140" width="128" height="40" fill="#ef4444" opacity="0.18"/>
  <text x="10" y="155" font-size="12" fill="#374151" font-weight="bold">Calendar</text>
  <text x="10" y="169" font-size="11" fill="#6b7280">for V-1</text>
  <text x="186" y="165" font-size="11" fill="#ffffff" text-anchor="middle">R1</text>
  <text x="474" y="165" font-size="11" fill="#ffffff" text-anchor="middle">R2</text>

  <line x1="218" y1="78" x2="218" y2="134" stroke="#dc2626" stroke-width="1.5" marker-end="url(#arrowA)"/>
  <line x1="346" y1="128" x2="346" y2="136" stroke="#16a34a" stroke-width="1.5" stroke-dasharray="3,3" marker-end="url(#arrowB)"/>

  <path d="M154 195 v-6 M282 195 v-6 M154 195 H282" stroke="#dc2626" stroke-width="1.5" fill="none"/>
  <text x="218" y="209" font-size="11" fill="#dc2626" text-anchor="middle">overlap</text>
  <line x1="282" y1="140" x2="282" y2="195" stroke="#16a34a" stroke-width="1.5" stroke-dasharray="3,3"/>
  <text x="282" y="209" font-size="11" fill="#16a34a" text-anchor="middle">boundary OK</text>

  <g font-size="12" fill="#374151" text-anchor="middle">
    <text x="122" y="236">Mon</text>
    <text x="186" y="236">Tue</text>
    <text x="250" y="236">Wed</text>
    <text x="314" y="236">Thu</text>
    <text x="378" y="236">Fri</text>
    <text x="442" y="236">Sat</text>
    <text x="506" y="236">Sun</text>
  </g>

  <defs>
    <marker id="arrowA" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0 0 L10 5 L0 10 z" fill="#dc2626"/>
    </marker>
    <marker id="arrowB" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0 0 L10 5 L0 10 z" fill="#16a34a"/>
    </marker>
  </defs>
</svg>
```

## Design Patterns Used

The pattern worth naming is the same one that fits the library, the Facade on `RentalService`, but the deeper structural idea here is not a GoF pattern at all: it is the time-interval model. `VehicleCalendar` is the data structure that makes the domain work, and the overlap predicate is the logic that makes it correct. Do not force a Strategy for pricing tiers or an Observer for notifications; neither exists in the requirements, and adding them invites the interviewer to ask "why?" with no good answer. The one place a Strategy could genuinely earn its keep, if pushed, is the pricing model, since per-type daily rates already live on the vehicle, and a future "weekly discount" would need a seam. Say that, show where the seam goes, and do not build it yet.

## Handling Edge Cases / Concurrency

The concurrency story is the same as inventory's, and it is mandatory here: two customers book the same car for the same weekend, both searches return the car, both call `book`, and without the `synchronized` both succeed. The fix is the atomic `tryAdd`, and the interview version of it is "the calendar serializes the check and the add, like a conditional insert in a database with a constraint on the date range." In a real system that is an exclusion constraint, `EXCLUDE USING gist (daterange(dates) WITH &&)`, and the candidate who can mention that has been in the field.

The edge cases beyond concurrency: a reservation where the return date is before or equal to the pickup date, which `overlaps` would happily consider free; the boundary case where one reservation ends exactly when another starts, which must be allowed and which the strict `isBefore` comparisons handle correctly; and a cancelled reservation, which the calendar must ignore, because a cancelled booking must free its window immediately. Each of these is a two-line guard or a comparison choice, and each is worth naming in the walkthrough.

## Common Mistakes

The most common mistake is the boolean availability field. `Vehicle.isAvailable` flipped to false when booked and true when returned. This cannot represent the future, which is the entire domain: a car can be booked next week and free tomorrow, and one boolean cannot hold both. Every candidate who draws it reaches the search question and has to invent the calendar on the spot.

The second mistake is searching by iterating reservations in the service and writing the overlap logic there. The overlap predicate belongs in `Reservation`, next to the dates it compares, and the availability query belongs in the calendar. Smearing the interval logic across the service means every follow-up ("what about a car with a maintenance window") forces a rewrite in a different place.

The third mistake is the check-then-add without a lock. The candidate who writes `if (calendar.isFree(...)) { reservations.add(...) }` in two separate lines has built the double-booking bug, and the interviewer will find it within one follow-up. The atomic `tryAdd` is not an optimization, it is the integrity of the system.

## Interview Perspective

A weak answer is a vehicle with `available` and a reservation that just holds a car and a customer. The search flow is a filter over cars by the flag, and the follow-up "book me a car for next week, I'll return it on Friday" produces a reservation that never checks anything. The system cannot answer the only question it exists to answer.

A strong answer says "availability is the calendar, the overlap rule lives on the reservation, and the book path is an atomic check-and-add on the calendar." Follow-ups to expect: "what if a car is returned damaged" (a condition check at return, which was cut from scope and would add a `MaintenanceEvent` to the calendar's set of blockers), "what if a customer books, then extends" (the extension is a new range and a new overlap check, same path), "how do you find the cheapest car for these dates" (sort the search result by rate, one line, which is exactly why search returns vehicles and not reservations). The strongest candidates volunteer the database exclusion constraint unprompted, because they know the interview version and the production version are the same idea.

## Knowledge Check

1. A car is booked from Monday to Wednesday. A customer asks for it Wednesday to Friday, and another asks for Tuesday to Thursday. Walk each request through `isFree` and state which, if either, succeeds.
2. Two customers search the same weekend, both see the same car, and both call `book` concurrently. Explain exactly where the race is and which method's lock prevents the double booking.
3. A reservation is cancelled after being confirmed. Explain what must happen to the calendar and why a naive `available = true` flag, if the design had one, would get this wrong.

## Key Takeaways

- Availability is intervals, not a boolean. A car free tomorrow and booked next week cannot be one flag.
- The overlap predicate lives on the reservation, where the dates are.
- The calendar is the gatekeeper: the atomic check-and-add is the difference between a booking system and a double-booking system.
- Search returns vehicles; book asks the calendar. Keep those responsibilities apart.
- The boundary case matters: end equals start must be allowed, and strict comparisons are what make it work.

## What's Next

The car rental system turned inventory into intervals and made the overlap check the integrity rule. The hotel booking system keeps the interval model almost exactly and changes the scale: rooms are interchangeable inventory, not specific cars, so availability becomes a count of rooms per night instead of a list of per-vehicle calendars.

---

This article explains how to design a car rental system around interval-based availability, where a vehicle's calendar decides whether a date range is free. It argues the boolean availability flag is the wrong model for anything rented over time.
