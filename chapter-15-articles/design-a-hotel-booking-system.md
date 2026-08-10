# Design a Hotel Booking System

## Learning Objectives

- Learn the inventory model that differs from the car rental's: rooms are interchangeable within a type, so availability is a count per night, not a per-room calendar.
- Design the night-by-night ledger that a booking decrements, and see why this model makes multi-night stays the natural unit instead of a special case.
- Handle the concurrency story that is now a count race: two bookings claiming the last room of a type on a shared night.

## Introduction

The hotel booking system looks like the car rental system wearing a uniform. Both hold something for a date range, both need an overlap rule, both get asked the same concurrency questions. The difference is the inventory granularity, and it is the entire point of the case study. A car is a specific car; a customer books that exact license plate. A hotel room is a category; the customer books "a deluxe king," and the front desk decides which physical room number they get. Interchangeable inventory changes the availability model from a list of calendars to a count per night, and that change cascades through the whole design. Interviewers ask this to see whether you notice that the same booking problem has two structurally different solutions, and whether you can pick the one the domain requires.

## Requirements Gathering

Functional requirements:

- The hotel has room types (standard, deluxe, suite), each with a total inventory and a nightly rate.
- A customer searches availability for a date range, gets room types with free capacity, and books one.
- A booking holds a room type, not a specific room, for its check-in and check-out dates.
- The system tracks bookings through confirmed, checked-in, checked-out, and cancelled states.
- The hotel can manage inventory, adding or removing rooms of a type.

Non-functional requirements:

- Availability per room type per night must be derivable quickly, and booking must be consistent under concurrent requests for the last room.
- A booking must never be accepted if it would push any night of its stay above the type's inventory.

Assumptions to state out loud: no specific-room assignment (rooms are interchangeable within a type, which is the whole premise), no pricing rules beyond nightly rate by type, no add-ons or taxes, no loyalty programs, and a booking is a single contiguous stay. Cut specific-room allocation and cut dynamic pricing. If you do not cut room assignment, you will spend the interview building a room-pairing algorithm that the requirements explicitly do not contain.

## Identifying Core Entities

The entity list has one deliberate difference from the car rental: no per-room calendar. That absence is the design decision.

| Entity | One-line responsibility |
| --- | --- |
| `RoomType` | A category with a name, a total inventory count, and a nightly rate. |
| `Booking` | A reservation of a room type for a check-in and check-out date. |
| `NightLedger` | The count of rooms of a type occupied on each future night. |
| `HotelInventory` | Holds all room types and their night ledgers, and answers capacity questions. |
| `BookingService` | The facade that searches and books against the inventory. |

There is no `Room` class. That is not an oversight, it is the answer. If a room is interchangeable within its type, the individual room is an implementation detail of the building, not of the booking system.

## Class Design

`RoomType` is metadata plus a capacity. It knows how many rooms of its type exist, and it knows its rate. It does not know who is staying.

```java
public class RoomType {
    private final String name;
    private int totalRooms;
    private final long nightlyRateInCents;

    public RoomType(String name, int totalRooms, long nightlyRateInCents) {
        this.name = name;
        this.totalRooms = totalRooms;
        this.nightlyRateInCents = nightlyRateInCents;
    }

    public void addRooms(int n) { totalRooms += n; }
    public int getTotalRooms() { return totalRooms; }
    public String getName() { return name; }
    public long getNightlyRateInCents() { return nightlyRateInCents; }
}
```

`Booking` is the reservation. It holds a room type, the dates, the guest, and its lifecycle status. The two interval methods, `nights()` and `touches`, are the whole arithmetic of the system.

```java
public class Booking {
    public enum Status { CONFIRMED, CHECKED_IN, CHECKED_OUT, CANCELLED }

    private final String id;
    private final RoomType roomType;
    private final String guestName;
    private final LocalDate checkIn;
    private final LocalDate checkOut;
    private Status status = Status.CONFIRMED;

    public Booking(String id, RoomType roomType, String guestName,
                   LocalDate checkIn, LocalDate checkOut) {
        this.id = id;
        this.roomType = roomType;
        this.guestName = guestName;
        this.checkIn = checkIn;
        this.checkOut = checkOut;
    }

    public List<LocalDate> nights() {
        return checkIn.datesUntil(checkOut).toList();
    }

    public void cancel() { status = Status.CANCELLED; }

    public RoomType getRoomType() { return roomType; }
    public LocalDate getCheckIn() { return checkIn; }
    public LocalDate getCheckOut() { return checkOut; }
    public Status getStatus() { return status; }
    public String getId() { return id; }
}
```

`NightLedger` is the model that makes the hotel different from the car rental. Instead of a list of reservation objects per vehicle, the hotel holds a count per night: how many rooms of this type are already committed on that night. A booking decrements every night it touches, and the capacity check is "is any night in this stay already at full."

```java
public class NightLedger {
    private final Map<LocalDate, Integer> occupied = new HashMap<>();
    private final int capacity;

    public NightLedger(int capacity) { this.capacity = capacity; }

    public boolean canAccommodate(List<LocalDate> nights) {
        for (LocalDate night : nights) {
            if (occupied.getOrDefault(night, 0) + 1 > capacity) {
                return false;
            }
        }
        return true;
    }

    public synchronized boolean tryCommit(List<LocalDate> nights) {
        if (!canAccommodate(nights)) {
            return false;
        }
        for (LocalDate night : nights) {
            occupied.merge(night, 1, Integer::sum);
        }
        return true;
    }

    public synchronized void release(List<LocalDate> nights) {
        for (LocalDate night : nights) {
            occupied.merge(night, -1, Integer::sum);
        }
    }

    public int freeOn(LocalDate date) {
        return capacity - occupied.getOrDefault(date, 0);
    }
}
```

`HotelInventory` holds a room type to its ledger. The search question is "how many nights in this range have capacity for this type," and the answer is a lookup per night, which is why the count model is fast: a month-long search is thirty lookups, not a scan of every booking ever made.

```java
public class HotelInventory {
    private final Map<String, RoomType> roomTypes = new ConcurrentHashMap<>();
    private final Map<String, NightLedger> ledgers = new ConcurrentHashMap<>();

    public void register(RoomType type) {
        roomTypes.put(type.getName(), type);
        ledgers.put(type.getName(), new NightLedger(type.getTotalRooms()));
    }

    public List<RoomType> searchAvailable(LocalDate checkIn, LocalDate checkOut) {
        List<LocalDate> nights = checkIn.datesUntil(checkOut).toList();
        return roomTypes.values().stream()
                .filter(t -> ledgers.get(t.getName()).canAccommodate(nights))
                .toList();
    }

    public int freeRooms(String typeName, LocalDate date) {
        return ledgers.get(typeName).freeOn(date);
    }
}
```

`BookingService.book` is the money operation. It checks the ledger for the whole stay and commits the whole stay, atomically. The unit of the check is the stay, not the night: a booking that fits night one but overflows night three must be rejected as a whole, because a partial booking is not a booking.

```java
public class BookingService {
    private final HotelInventory inventory;

    public BookingService(HotelInventory inventory) { this.inventory = inventory; }

    public Optional<Booking> book(RoomType type, String guestName,
                                  LocalDate checkIn, LocalDate checkOut) {
        NightLedger ledger = getLedger(type);
        List<LocalDate> nights = checkIn.datesUntil(checkOut).toList();
        if (!ledger.tryCommit(nights)) {
            return Optional.empty();
        }
        Booking booking = new Booking(UUID.randomUUID().toString(), type, guestName, checkIn, checkOut);
        return Optional.of(booking);
    }

    public void cancel(Booking booking) {
        booking.cancel();
        getLedger(booking.getRoomType())
                .release(booking.getCheckIn().datesUntil(booking.getCheckOut()).toList());
    }

    private NightLedger getLedger(RoomType type) {
        return inventory.getLedger(type.getName());
    }
}
```

Diagram: the count-per-night ledger for one room type. Each booking decrements every night it touches, and the new booking commits the whole stay only if no night would exceed capacity.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 860 410" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <rect width="860" height="410" fill="#ffffff"/>
  <text x="430" y="34" text-anchor="middle" font-size="20" font-weight="bold" fill="#1f2937">Count-per-night ledger for "Deluxe" (capacity 5)</text>

  <g font-size="14" font-weight="bold" fill="#374151" text-anchor="middle">
    <rect x="150" y="80" width="100" height="32" rx="6" fill="#eef2ff" stroke="#c7d2fe"/>
    <text x="200" y="100">Mon</text>
    <rect x="260" y="80" width="100" height="32" rx="6" fill="#eef2ff" stroke="#c7d2fe"/>
    <text x="310" y="100">Tue</text>
    <rect x="370" y="80" width="100" height="32" rx="6" fill="#eef2ff" stroke="#c7d2fe"/>
    <text x="420" y="100">Wed</text>
    <rect x="480" y="80" width="100" height="32" rx="6" fill="#eef2ff" stroke="#c7d2fe"/>
    <text x="530" y="100">Thu</text>
    <rect x="590" y="80" width="100" height="32" rx="6" fill="#eef2ff" stroke="#c7d2fe"/>
    <text x="640" y="100">Fri</text>
  </g>

  <g font-size="13">
    <text x="30" y="148" fill="#374151">B1 Mon→Wed</text>
    <rect x="150" y="130" width="210" height="30" rx="6" fill="#bfdbfe" stroke="#3b82f6"/>
    <text x="255" y="150" text-anchor="middle" fill="#1e40af" font-weight="bold">nights: Mon, Tue</text>
    <text x="30" y="192" fill="#374151">B2 Tue→Thu</text>
    <rect x="260" y="174" width="210" height="30" rx="6" fill="#bfdbfe" stroke="#3b82f6"/>
    <text x="365" y="194" text-anchor="middle" fill="#1e40af" font-weight="bold">nights: Tue, Wed</text>
    <text x="30" y="236" fill="#374151">B3 Wed→Fri</text>
    <rect x="370" y="218" width="210" height="30" rx="6" fill="#bfdbfe" stroke="#3b82f6"/>
    <text x="475" y="238" text-anchor="middle" fill="#1e40af" font-weight="bold">nights: Wed, Thu</text>
  </g>

  <g font-size="13" text-anchor="middle">
    <text x="430" y="292" font-weight="bold" fill="#374151">Occupied count per night</text>
    <rect x="150" y="300" width="100" height="36" rx="6" fill="#f3f4f6" stroke="#d1d5db"/>
    <text x="200" y="323" font-weight="bold" fill="#374151">1</text>
    <rect x="260" y="300" width="100" height="36" rx="6" fill="#f3f4f6" stroke="#d1d5db"/>
    <text x="310" y="323" font-weight="bold" fill="#374151">2</text>
    <rect x="370" y="300" width="100" height="36" rx="6" fill="#fef3c7" stroke="#f59e0b"/>
    <text x="420" y="323" font-weight="bold" fill="#92400e">3</text>
    <rect x="480" y="300" width="100" height="36" rx="6" fill="#f3f4f6" stroke="#d1d5db"/>
    <text x="530" y="323" font-weight="bold" fill="#374151">2</text>
    <rect x="590" y="300" width="100" height="36" rx="6" fill="#f3f4f6" stroke="#d1d5db"/>
    <text x="640" y="323" font-weight="bold" fill="#374151">0</text>
  </g>

  <g font-size="13">
    <text x="30" y="392" fill="#374151">New booking Tue→Thu</text>
    <rect x="260" y="374" width="210" height="30" rx="6" fill="none" stroke="#16a34a" stroke-width="2" stroke-dasharray="6 4"/>
    <text x="365" y="394" text-anchor="middle" fill="#15803d" font-weight="bold">check Tue (2+1) &amp; Wed (3+1) ≤ 5</text>
    <text x="520" y="392" fill="#15803d" font-weight="bold">COMMIT whole stay ✓</text>
  </g>
</svg>
```

The design is complete in about 150 lines, and the absence of a `Room` class is visible in the line count. Every method answers a question a hotel front desk actually asks.

## Design Patterns Used

The honest answer here is the same as the car rental's, with one addition. No classic pattern is doing heavy lifting; the structural idea is the count-per-night ledger, which is the aggregated version of the car rental's per-vehicle calendar. If pushed for a pattern, the interesting comparison to offer is that this is an event-sourcing-flavored aggregate in miniature: the ledger is the projected count of bookings, and it can be rebuilt by replaying bookings. Do not add an Observer for confirmation emails or a Strategy for pricing. The requirements have one pricing rule, nightly by type, and the seam for more pricing sits on `RoomType`. Name it, do not build it.

## Handling Edge Cases / Concurrency

The concurrency story is the count race, and it is the strongest version of the inventory race yet. Two guests each book the last standard room for the same night. Both ask the ledger, both see 49 of 50 occupied, both commit. The `synchronized` on `tryCommit` closes the gap: the check and the increments are one unit, so the second caller sees 50 and is rejected. In a real database this is a row lock or an atomic `UPDATE ... WHERE occupied < capacity` on the night row, and naming that shows you know the production version.

The edge cases beyond the race: the multi-night stay, which is not a special case because the ledger commits every night in one `tryCommit`; the cancellation, which must release every night of the stay, and which is why `release` takes the same `nights()` list the commit took; and the check-out boundary, where a guest leaving on the same day another arrives is legal because `datesUntil` excludes the check-out date, so no night double-counts. Each of these falls out of the model rather than requiring a guard, which is the sign the model is right.

## Common Mistakes

The most common mistake is importing the car rental model wholesale: a `Room` class with its own calendar, and a booking that holds a specific room. That design works only if the hotel assigns specific rooms at booking time, which is not how hotels work, and the follow-up "a guest books a deluxe, any deluxe will do" produces a booking that points at a room the guest may not get. The interchangeable inventory is the domain, and the model must say so.

The second mistake is checking and committing one night at a time. `if (free(today)) book(today); if (free(tomorrow)) book(tomorrow);` is a design that accepts partial stays, then has to unwind when the third night is full, and the unwinding is where the concurrency bugs live. The stay is the atomic unit.

The third mistake is holding a per-room occupancy list and scanning it for every search. That is the car rental calendar at hotel scale, and it makes "how many deluxe are free on July 14" a linear scan over every booking. The count ledger turns that question into one map lookup, and choosing the count model is not an optimization, it is the correct abstraction for interchangeable inventory.

## Interview Perspective

A weak answer is the car rental design renamed: `Room` with an `available` flag, bookings against specific rooms, and no notion of capacity. The interviewer asks "how many deluxe rooms are free on Friday" and the candidate has to count booleans across the whole room list, if they can answer at all.

A strong answer says "rooms are interchangeable within a type, so availability is a count per night, and a booking commits the whole stay against the ledger atomically." That sentence is the case study. Follow-ups to expect: "what if the hotel has 10 deluxe and 12 are already booked" (impossible, the ledger rejects the twelfth, which is the capacity invariant), "what if a guest checks out early" (a release of the remaining nights, same `release` path, which is honest because the reservation model cannot know about early check-outs until they happen), "how do you add a pricing rule for weekends" (a seam on the rate calculation, which the strong candidate named and did not build). The strongest candidates bring up the count race and the `UPDATE ... WHERE occupied < capacity` phrasing themselves, because they have seen this exact bug in production.

## Knowledge Check

1. A type has 5 rooms. One booking covers Monday to Wednesday, another covers Tuesday to Thursday, and a third covers Wednesday to Friday. Walk each through the ledger and state the occupied count for Tuesday and Wednesday after all three.
2. Two guests simultaneously book the last of 5 rooms for a single night. Trace both calls through `tryCommit` and explain which succeeds, which fails, and what the ledger shows for that night afterward.
3. The car rental design uses a per-room calendar of reservations. Explain why that model is the wrong shape for a hotel, and what specifically breaks when you try to answer "how many deluxe are free on Friday" with it.

## Key Takeaways

- Interchangeable inventory means availability is a count per night, not a per-room calendar. No `Room` class needed.
- A booking commits every night of the stay in one atomic operation. Partial stays are not a state the system can be in.
- The count race on the last room is the concurrency story, and `synchronized` here, a conditional update in the database, closes it.
- Cancellation releases the same night list the commit consumed. Symmetry is correctness.
- The count ledger turns "free on a date" into a lookup, which is why this model scales where per-room scanning does not.

## What's Next

The hotel booking system traded per-item calendars for count-per-night ledgers, and the atomic stay became the unit of correctness. The URL shortener leaves all of this behind: there is no inventory and no time range, only a mapping problem, and the entire design is about how you generate and store the keys.

---

This article explains how to design a hotel booking system around a count-per-night ledger, since rooms are interchangeable within a type. It argues there is deliberately no Room class, and the whole-stay commit is what makes partial bookings impossible.
