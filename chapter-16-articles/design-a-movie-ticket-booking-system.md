# Design a Movie Ticket Booking System: Handling Concurrency and Locking

## Learning Objectives

- Design the seat as the unit of contention and decide, explicitly, how long a seat can be held and what happens when the holder abandons it.
- Compare the two locking strategies, pessimistic and optimistic, and know what each one costs in throughput and complexity.
- Model the booking as a state machine, REQUESTED, LOCKED, CONFIRMED, CANCELLED, and make every transition a guard against a concurrent race.

## Introduction

The movie ticket booking system is the case study this chapter was building toward. Every concurrency problem from the earlier cases shows up here in one system: the double-sale race from inventory, the interval conflict from bookings, the atomic check-and-apply from the car rental, all of them concentrated on one object, the seat. A seat can be sold once. Two people cannot both get the same seat for the same show, and the hard part is not the rule, it is the timing: how long between "this seat looks free" and "this seat is yours" should the system hold it? Hold too long and one abandoned checkout blocks everyone; release too early and two people pay for the same seat. There is no correct answer, only a position, and the interview is whether you can take one and defend its cost.

## Requirements Gathering

Functional requirements:

- A show is a movie at a theater at a time, with a fixed set of seats in a hall layout.
- A user selects seats, the system reserves them for a limited window while the user completes the payment, and confirms the booking on success.
- A user can see which seats are free, held, and sold for a show.
- A booking that is not paid within the hold window is released.
- The same seat cannot be confirmed to two different users for the same show.

Non-functional requirements:

- Concurrent bookings of the same seat must resolve to exactly one winner.
- The hold window must balance fairness to a user mid-checkout against throughput for everyone else.

Assumptions to state out loud: no seat-preference pricing or VIP rows, no group bookings with seat adjacency requirements, no refunds after confirmation, and the hold window is a fixed duration per booking, not dynamic. Cut group adjacency and cut refunds. The interviewer wants the seat-lock model, and both cuts keep it from turning into a constraint-satisfaction problem.

## Identifying Core Entities

The entity list is the movie theater as data, and the booking as the state machine.

| Entity | One-line responsibility |
| --- | --- |
| `Show` | A movie at a theater at a time, owning its hall's seat set. |
| `Seat` | A physical chair in a hall, with a state for a given show. |
| `SeatState` | The per-show lifecycle of a seat: FREE, LOCKED, SOLD. |
| `Booking` | A user's claim on a set of seats, moving through REQUESTED, LOCKED, CONFIRMED, CANCELLED. |
| `BookingService` | The facade where locking, confirming, and releasing happen. |
| `HoldRegistry` | Tracks which seats are held and by which booking, with the hold deadline. |

The interesting entity is `HoldRegistry`, because it is the object that owns the contention. The `Booking` is a state machine, but the `HoldRegistry` is the thing that decides whether a lock can happen at all.

## Class Design

Start with the seat state. A seat is a place in a hall, and its state for a show is what matters. The state transition is the whole concurrency story, so the transition method is where the guard lives.

```java
public enum SeatState { FREE, LOCKED, SOLD }

public class Seat {
    private final String seatId;
    private final int row;
    private final int number;

    public Seat(String seatId, int row, int number) {
        this.seatId = seatId;
        this.row = row;
        this.number = number;
    }

    public String getSeatId() { return seatId; }
    public int getRow() { return row; }
    public int getNumber() { return number; }
}
```

`Booking` is the state machine. The transitions are the design: a booking starts REQUESTED, a lock makes it LOCKED, payment makes it CONFIRMED, and either expiry or user action cancels it. Every transition must be legal, and the legality is enforced by the guard methods, not by hope.

```java
public class Booking {
    public enum Status { REQUESTED, LOCKED, CONFIRMED, CANCELLED }

    private final String bookingId;
    private final String userId;
    private final String showId;
    private final List<Seat> seats;
    private Status status = Status.REQUESTED;
    private final Instant holdExpiresAt;

    public Booking(String bookingId, String userId, String showId, List<Seat> seats,
                   Instant holdExpiresAt) {
        this.bookingId = bookingId;
        this.userId = userId;
        this.showId = showId;
        this.seats = seats;
        this.holdExpiresAt = holdExpiresAt;
    }

    public boolean lock() {
        if (status != Status.REQUESTED) return false;
        status = Status.LOCKED;
        return true;
    }

    public boolean confirm() {
        if (status != Status.LOCKED) return false;
        status = Status.CONFIRMED;
        return true;
    }

    public boolean cancel() {
        if (status != Status.CONFIRMED) return false;
        status = Status.CANCELLED;
        return true;
    }

    public boolean hasExpired() {
        return Instant.now().isAfter(holdExpiresAt);
    }
}
```

`HoldRegistry` is where the locking happens, and it is where the case study earns its subtitle. The design question is which locking strategy to model. The pessimistic version locks the seat's state row for the duration of the hold: the registry stores seat to booking, and a second attempt at the same seat fails while the first hold is live. The optimistic version does not lock anything up front; the seat carries a version counter, and the confirm step checks that the version is unchanged since the hold was taken.

The pessimistic version, in its honest interview form:

```java
public class HoldRegistry {
    private final Map<String, String> seatToBooking = new ConcurrentHashMap<>();
    private final Map<String, Booking> bookings = new ConcurrentHashMap<>();

    public synchronized boolean tryLock(Booking booking) {
        for (Seat seat : booking.getSeats()) {
            if (seatToBooking.containsKey(seat.getSeatId())) {
                return false;
            }
        }
        for (Seat seat : booking.getSeats()) {
            seatToBooking.put(seat.getSeatId(), booking.getBookingId());
        }
        bookings.put(booking.getBookingId(), booking);
        return true;
    }

    public synchronized void release(String bookingId) {
        Booking booking = bookings.remove(bookingId);
        if (booking == null) return;
        for (Seat seat : booking.getSeats()) {
            seatToBooking.remove(seat.getSeatId());
        }
    }
}
```

The `synchronized` around the whole check-and-put is the same atomicity you have seen three times in this chapter, but here the held resource is a seat, the hold lasts minutes instead of microseconds, and the window between check and confirm is the whole problem.

The optimistic version changes where the check happens. The lock step records the seat's version; the confirm step verifies no one else changed it in between. Two users can both hold the same seat optimistically, but only one confirm succeeds.

```java
public class SeatVersion {
    private final String seatId;
    private volatile long version;

    public boolean tryIncrement() {
        // compare-and-swap style: caller passes the version it saw
        return false; // replaced below by the service flow
    }
}
```

The interview answer for optimistic is usually prose rather than code: "the seat row carries a version, the confirm is `UPDATE seats SET version = version + 1 WHERE seat_id = ? AND version = ?`, and if the affected row count is zero, someone else won." That one sentence, with the affected-rows check, is the optimistic locking answer.

`BookingService` wires it together, with the expiry sweep that turns the "abandoned checkout" problem into a background responsibility. The sweep is what makes the hold window meaningful: it does not just let a hold time out conceptually, it actively returns the seats to the pool.

```java
public class BookingService {
    private final HoldRegistry registry;
    private final Map<String, Show> shows;

    public Optional<Booking> bookSeats(String userId, String showId, List<Seat> seats) {
        Booking booking = new Booking(UUID.randomUUID().toString(), userId, showId, seats,
                Instant.now().plus(Duration.ofMinutes(5)));
        if (!registry.tryLock(booking)) {
            return Optional.empty();
        }
        return Optional.of(booking);
    }

    public boolean confirmPayment(String bookingId) {
        Booking booking = registry.get(bookingId);
        if (booking == null || booking.hasExpired()) {
            registry.release(bookingId);
            return false;
        }
        if (!booking.confirm()) {
            return false;
        }
        return true;
    }

    public void sweepExpiredHolds() {
        for (Booking booking : registry.allBookings()) {
            if (booking.hasExpired() && booking.getStatus() == Booking.Status.LOCKED) {
                registry.release(booking.getBookingId());
            }
        }
    }
}
```

Diagram: the seat is the unit of contention. Top, the seat lifecycle with the hold window and the sweep that releases abandoned holds. Bottom, the race that resolves to exactly one winner.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 360" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="ah3" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
  </defs>
  <rect width="900" height="360" fill="#ffffff"/>

  <text x="450" y="26" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">The seat is the unit of contention</text>
  <text x="450" y="48" text-anchor="middle" font-size="14" fill="#6b7280">FREE → LOCKED (5-minute hold) → SOLD</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ah3)">
    <line x1="270" y1="106" x2="346" y2="106"/>
    <line x1="540" y1="106" x2="616" y2="106"/>
    <polyline points="445,134 445,178 175,178 175,138"/>
  </g>
  <g stroke="#dc2626" stroke-width="1.8" stroke-dasharray="6 4" fill="none" marker-end="url(#ah3)">
    <polyline points="445,134 445,178 175,178 175,138"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle">
    <rect x="80" y="78" width="190" height="56" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
    <text x="175" y="111" fill="#1e3a8a">FREE</text>
    <rect x="350" y="78" width="190" height="56" rx="8" fill="#fef3c7" stroke="#f59e0b"/>
    <text x="445" y="105" fill="#92400e">LOCKED</text>
    <text x="445" y="124" font-size="12" fill="#b45309">5-min hold</text>
    <rect x="620" y="78" width="190" height="56" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="715" y="111" fill="#14532d">SOLD</text>
  </g>

  <g font-size="12.5" fill="#475569" text-anchor="middle">
    <text x="308" y="96">tryLock — one racer wins</text>
    <text x="578" y="96">confirmPayment</text>
    <text x="310" y="208" fill="#b91c1c">hold expires / late payment → sweep releases back to FREE</text>
  </g>

  <text x="450" y="232" text-anchor="middle" font-size="16" font-weight="bold" fill="#1f2937">The race: two users, one last seat</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ah3)">
    <line x1="200" y1="318" x2="246" y2="318"/>
    <line x1="430" y1="318" x2="476" y2="318"/>
    <line x1="660" y1="318" x2="706" y2="318"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12.5" fill="#1f2937">
    <rect x="20" y="270" width="180" height="70" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="110" y="296" text-anchor="middle" font-weight="bold" fill="#334155">1. A and B select</text>
    <text x="110" y="314" text-anchor="middle">the same last</text>
    <text x="110" y="330" text-anchor="middle">seat for the show</text>
    <rect x="250" y="270" width="180" height="70" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="340" y="296" text-anchor="middle" font-weight="bold" fill="#334155">2. Both call</text>
    <text x="340" y="314" text-anchor="middle">bookSeats(seat)</text>
    <text x="340" y="330" text-anchor="middle"></text>
    <rect x="480" y="270" width="180" height="70" rx="8" fill="#fef3c7" stroke="#f59e0b"/>
    <text x="570" y="296" text-anchor="middle" font-weight="bold" fill="#92400e">3. tryLock is atomic</text>
    <text x="570" y="314" text-anchor="middle">A wins → LOCKED</text>
    <text x="570" y="330" text-anchor="middle">B gets empty → reselect</text>
    <rect x="710" y="270" width="180" height="70" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="800" y="296" text-anchor="middle" font-weight="bold" fill="#14532d">4. A confirms → SOLD</text>
    <text x="800" y="314" text-anchor="middle">or hold expires →</text>
    <text x="800" y="330" text-anchor="middle">sweep → back to FREE</text>
  </g>

</svg>
```

The walkthrough of the race: two users select the same last seat, both book, `tryLock` lets exactly one through, the loser sees an empty result and the UI offers a different seat. That is the whole system working.

## Design Patterns Used

The two patterns that genuinely matter here are the two locking strategies, which are not GoF patterns but are real design choices: pessimistic locking, reserve the resource up front, and optimistic locking, verify at commit. The case study is literally about choosing between them, so name them, compare them, and pick one with a reason. The pessimistic model fits a movie theater because contention on a single show is real but bounded, the hold is short, and the simplicity of "the seat is simply not available" is worth more than the extra throughput of optimistic retries. The optimistic model fits a system where contention is rare and retries are cheap. There is no Observer needed for the seat map UI; a poll or a push on booking events is enough. And resist the Command pattern for the booking flow; a service with three methods is the honest shape.

## Handling Edge Cases / Concurrency

The edge cases here are the interview, so go through them deliberately. The abandoned checkout: a user locks a seat and walks away. The hold expires, and the sweep releases it. Without the sweep, a weekend of abandoned checkouts locks the entire hall, and naming the sweep unprompted is a senior signal.

The double-confirm race: the optimistic strategy lets two users hold the same seat, and both try to pay. The affected-rows check on the conditional update decides the winner and the loser's payment must be refused or refunded. The loser's refund is a payment-systems concern, but the "who won" decision is here.

The partial-seat case: a booking with three seats where two lock and the third is taken. The design returns false on the third and releases nothing, which is the wrong behavior: `tryLock` must be all-or-nothing, which is why the pessimistic version checks every seat before putting any, so a partial hold is never left behind. That ordering, check all, then put all, is the atomicity the whole lock depends on.

The expiry boundary: a user completes payment exactly as the hold expires. The confirm path checks `hasExpired` first, so a late payment loses, and the seat was already returned to the pool by the sweep. The rule to state is that expiry is checked at confirm time, not assumed, because the sweep and the confirm race each other.

## Common Mistakes

The most common mistake is a boolean `isBooked` on the seat, flipped at booking and back at release, with no hold concept at all. The user who starts a checkout loses the seat the moment someone else clicks it, and there is no window, no sweep, and no reason to believe the seat map. The hold window is the product feature, not an optional extra.

The second mistake is the non-atomic lock: check each seat, then put each seat, in two loops with a real possibility of another thread interleaving between them. The `synchronized` on the whole method, or the database row locks underneath it, is what makes the check-then-put a single unit. A candidate who writes the two loops without the lock has built the double-sell.

The third mistake is deciding the locking strategy without stating its cost. Saying "I'll use optimistic locking" and stopping is a non-answer. The sentence must be "optimistic, because contention is rare here, and the retry on confirm failure is a UX detail, not a correctness problem" or the pessimistic version with its own cost. The strategy is a trade-off, and stating it as one is the whole point.

## Interview Perspective

A weak answer is a `Seat` with `booked` and a `Booking` that flips it. The interviewer asks "what if two people pick the same seat at the same moment" and the answer is "well, we check if it's booked first," with no mechanism for the race. The hold window, the sweep, and the locking strategy are all absent, because the candidate never decided what "in progress" means.

A strong answer says "the seat is the unit of contention, the hold is the product feature, the lock is pessimistic because contention is real but short, and the sweep is what prevents abandoned holds from bricking the hall." The follow-ups then land cleanly. "How long do you hold the lock" (five minutes, stated as a product trade-off, not a guess). "What if the payment takes longer than the hold" (the confirm checks expiry and fails, the user rebooks, which is the honest cost of the short hold). "How do you pick the winning user" (the atomic lock decides, and the loser's UI re-selects, which is the walkthrough). The strongest candidates compare both locking strategies without prompting and state which one they chose and why, because that is the case study's actual deliverable.

## Knowledge Check

1. Two users simultaneously book the last seat of a show under the pessimistic strategy. Trace both calls through `tryLock` and describe exactly what each user's UI observes.
2. Explain the difference between the hold window and the seat state, and what would break if the sweep were removed and holds only expired on paper.
3. Under optimistic locking, two users hold the same seat and both confirm within milliseconds. Walk through the conditional update and state which confirm succeeds and what the loser's payment flow must do.

## Key Takeaways

- The seat is the unit of contention, and its hold window is the product feature.
- Pessimistic locking reserves up front, optimistic verifies at commit. Pick one and price its cost out loud.
- The lock is check-every-seat-then-put-every-seat, all or nothing, as one atomic unit.
- The expiry sweep is what makes abandoned checkouts survivable, and expiry is re-checked at confirm time.
- The loser's flow is part of the design: empty result, UI re-select, no half-locked seats behind them.

## What's Next

The movie booking system made the seat the unit of contention and the hold window the product. The e-commerce cart keeps the seat, renames it to a stock unit, and widens the flow: the cart is not a lock, it is a wish list, and the lock moves to the order placement, which changes where the concurrency lives.

---

This article explains how to design a movie ticket booking system around the seat as the unit of contention and the hold window that makes checkouts fair. Its point of view is that the locking strategy and its stated cost are the entire design.
