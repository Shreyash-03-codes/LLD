# Design an Elevator System

## Learning Objectives

- Build a system where the interesting logic lives in a controller that makes decisions, not in data holders.
- Understand why a priority queue keyed by direction is the heart of elevator scheduling, and what happens when you skip it.
- See how a State object for a moving thing beats a pile of booleans, without reaching for a heavyweight pattern.

## Introduction

The elevator is the first LLD case study where a naive class model passes you at a glance and fails you the moment you trace a flow. A parking lot's logic is nearly stateless: a car comes, a car goes. An elevator is the opposite. It has position, direction, a set of pending stops, and rules about which stop to serve next, and those rules are what the interview is actually testing. Interviewers ask this because it is the cleanest way to see whether you can separate "what the elevator is" from "how the elevator decides." Most candidates can draw the first half. Very few can reason about the second.

## Requirements Gathering

Functional requirements:

- An elevator has a current floor and a direction, idle or moving up or down.
- Passengers press buttons inside the cabin to request a floor; people on each floor press up or down arrows to call the elevator.
- The elevator stops at requested floors in the direction it is traveling, then reverses at the end of the pending requests.
- The cabin door opens only when the elevator is stopped and level with a floor.
- The system reports its current floor and direction to the display panels on each floor.

Non-functional requirements:

- The controller must never starve a request; every pressed button eventually gets served.
- Scheduling decisions should be simple enough to reason about. Nobody wants a "clever" scheduler that a maintenance engineer cannot predict.

Assumptions to state out loud: a single elevator (multi-elevator dispatch is the classic follow-up, not the base case), no emergency or overload handling, doors always open and close instantly with no hold timer. These cuts keep the interview about scheduling, which is the point. If you try to model door timers and emergency braking, you will run out of time before the controller exists.

## Identifying Core Entities

The entity list is short, and the weight is distributed unevenly.

| Entity | One-line responsibility |
| --- | --- |
| `Elevator` | The physical cabin: current floor, direction, door state, and the actual move commands. |
| `ElevatorController` | The brain: receives requests, decides the next stop, and drives the elevator. |
| `Request` | A floor plus the direction or intent behind it, from the cabin panel or from a floor button. |
| `Building` | Owns the controller and the external floor buttons. |

The asymmetry is deliberate. `Elevator` knows how to move. `ElevatorController` knows where to go. If you merge them, the follow-up ("now add a second elevator") becomes a rewrite. If you keep them apart, it becomes the controller owning a list.

## Class Design

Start with `Request`. A request is a destination floor. Where it came from matters for scheduling, but the object itself is just the target. Keep a flag for whether it was pressed inside the cabin, because cabin requests always get served regardless of direction; floor requests only get served if the call direction matches travel.

```java
public class Request {
    private final int floor;
    private final Direction direction; // direction of travel, or NONE for cabin requests

    public Request(int floor, Direction direction) {
        this.floor = floor;
        this.direction = direction;
    }

    public int getFloor() { return floor; }
    public Direction getDirection() { return direction; }
}
```

`Direction` is an enum with two real values and an idle state. The idle state is a source of bugs if you model it as null, so make it explicit.

```java
public enum Direction {
    UP, DOWN, IDLE
}
```

`Elevator` is the machine. It tracks where it is, what it is doing, and whether the door is open. The movement methods are the meat: `moveOneFloor()` is the primitive the controller calls. In a simulation the elevator moves one floor at a time; in a real system the controller sends a target floor to the motor. Either way, the elevator never decides its own destination.

```java
public class Elevator {
    private int currentFloor;
    private Direction direction = Direction.IDLE;
    private boolean doorOpen;

    public Elevator(int initialFloor) {
        this.currentFloor = initialFloor;
    }

    public void moveOneFloor(Direction direction) {
        this.direction = direction;
        this.currentFloor += direction == Direction.UP ? 1 : -1;
        this.doorOpen = false;
    }

    public void openDoor() {
        doorOpen = true;
    }

    public void stop() {
        direction = Direction.IDLE;
    }

    public int getCurrentFloor() { return currentFloor; }
    public Direction getDirection() { return direction; }
}
```

Now the controller. This is where the design lives or dies. The standard approach is the SCAN algorithm (also called the elevator algorithm): keep two sets of pending floors, one for stops in the up direction and one for the down direction. While moving up, serve only floors above, in ascending order. When nothing is left above, reverse and serve the down set. The trick that makes this correct is that cabin requests always go into the set matching their target direction relative to current position, and floor calls go into the set matching the called direction.

Two `TreeSet<Integer>`s give you the ordering for free. This is the whole point of the case study: the data structure is the scheduling algorithm.

```java
import java.util.NavigableSet;
import java.util.TreeSet;

public class ElevatorController {
    private final Elevator elevator;
    private final NavigableSet<Integer> upStops = new TreeSet<>();
    private final NavigableSet<Integer> downStops = new TreeSet<>();

    public ElevatorController(Elevator elevator) {
        this.elevator = elevator;
    }

    public void addRequest(Request request) {
        int floor = request.getFloor();
        if (floor == elevator.getCurrentFloor()) {
            return; // already here
        }
        if (request.getDirection() == Direction.UP
                || (request.getDirection() == Direction.NONE && floor > elevator.getCurrentFloor())) {
            upStops.add(floor);
        } else {
            downStops.add(floor);
        }
    }

    public void run() {
        while (!upStops.isEmpty() || !downStops.isEmpty()) {
            if (elevator.getDirection() == Direction.DOWN) {
                serveDown();
            } else {
                serveUp();
            }
        }
        elevator.stop();
    }

    private void serveUp() {
        while (!upStops.isEmpty()) {
            int next = upStops.first();
            if (next < elevator.getCurrentFloor()) {
                upStops.pollFirst(); // stale stop, ignore
                continue;
            }
            moveTo(next);
            upStops.pollFirst();
        }
        elevator.stop();
        // everything up is served; sweep down requests if any
    }

    private void moveTo(int floor) {
        while (elevator.getCurrentFloor() != floor) {
            elevator.moveOneFloor(floor > elevator.getCurrentFloor() ? Direction.UP : Direction.DOWN);
        }
        elevator.stop();
        elevator.openDoor();
    }
}
```

Diagram: the SCAN algorithm in action — a car on floor 3 heading down with a down-stop at floor 1, and a floor-4 up call that must wait for the return trip.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 920 420" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
    <marker id="arrG" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#16a34a"/>
    </marker>
    <marker id="arrR" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#dc2626"/>
    </marker>
  </defs>
  <rect width="920" height="420" fill="#ffffff"/>

  <text x="460" y="26" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">SCAN scheduling — two direction-keyed sorted sets are the algorithm</text>

  <g stroke="#94a3b8" stroke-width="1.5">
    <line x1="120" y1="120" x2="210" y2="120"/>
    <line x1="120" y1="156" x2="210" y2="156"/>
    <line x1="120" y1="192" x2="210" y2="192"/>
    <line x1="120" y1="228" x2="210" y2="228"/>
    <line x1="120" y1="264" x2="210" y2="264"/>
    <line x1="120" y1="300" x2="210" y2="300"/>
    <line x1="120" y1="336" x2="210" y2="336"/>
    <line x1="120" y1="372" x2="210" y2="372"/>
    <line x1="120" y1="120" x2="120" y2="372"/>
    <line x1="210" y1="120" x2="210" y2="372"/>
  </g>
  <g font-size="12.5" fill="#475569" text-anchor="end">
    <text x="112" y="126">8F</text>
    <text x="112" y="162">7F</text>
    <text x="112" y="198">6F</text>
    <text x="112" y="234">5F</text>
    <text x="112" y="270">4F</text>
    <text x="112" y="306">3F</text>
    <text x="112" y="342">2F</text>
    <text x="112" y="378">1F</text>
  </g>

  <g stroke="#3b82f6" stroke-width="1.8" fill="none" marker-end="url(#arr)">
    <line x1="167" y1="336" x2="167" y2="352"/>
  </g>
  <g font-size="12" fill="#1d4ed8">
    <text x="178" y="350">car moving DOWN</text>
  </g>

  <rect x="132" y="300" width="70" height="36" rx="6" fill="#dbeafe" stroke="#3b82f6"/>
  <text x="167" y="323" text-anchor="middle" font-size="12" font-weight="bold" fill="#1e3a8a">car</text>

  <g stroke="#16a34a" stroke-width="2.4" fill="none" marker-end="url(#arrG)">
    <line x1="236" y1="292" x2="236" y2="276"/>
  </g>
  <text x="250" y="282" font-size="12.5" fill="#15803d" font-weight="bold">4F up call</text>

  <g stroke="#dc2626" stroke-width="2.4" fill="none" marker-end="url(#arrR)">
    <line x1="236" y1="366" x2="236" y2="378"/>
  </g>
  <text x="250" y="376" font-size="12.5" fill="#b91c1c" font-weight="bold">1F down call</text>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12.5" fill="#1f2937">
    <rect x="300" y="130" width="250" height="88" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="425" y="156" text-anchor="middle" font-weight="bold" fill="#14532d">upStops — ascending</text>
    <text x="425" y="186" text-anchor="middle" font-size="22" font-weight="bold" fill="#15803d">{ 4 }</text>
    <text x="425" y="208" text-anchor="middle" font-size="11.5" fill="#166534">floor-4 UP call lands here</text>

    <rect x="580" y="130" width="250" height="88" rx="8" fill="#fee2e2" stroke="#dc2626"/>
    <text x="705" y="156" text-anchor="middle" font-weight="bold" fill="#7f1d1d">downStops — ascending</text>
    <text x="705" y="186" text-anchor="middle" font-size="22" font-weight="bold" fill="#b91c1c">{ 1 }</text>
    <text x="705" y="208" text-anchor="middle" font-size="11.5" fill="#991b1b">floor-1 DOWN call lands here</text>
  </g>
  <text x="30" y="296" font-size="14" font-weight="bold" fill="#1f2937">The controller — decide, then move</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#arr)">
    <line x1="250" y1="348" x2="296" y2="348"/>
    <line x1="510" y1="348" x2="556" y2="348"/>
    <line x1="665" y1="376" x2="665" y2="396"/>
    <line x1="665" y1="396" x2="145" y2="396"/>
    <line x1="145" y1="396" x2="145" y2="376"/>
  </g>
  <text x="410" y="392" text-anchor="middle" font-size="12" fill="#475569">reverse → serve the other set; stop when both empty</text>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12" fill="#1f2937">
    <rect x="40" y="320" width="210" height="56" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="145" y="343" text-anchor="middle" font-weight="bold" fill="#334155">run(): while upStops or</text>
    <text x="145" y="358" text-anchor="middle" font-weight="bold" fill="#334155">downStops non-empty</text>
    <rect x="300" y="320" width="210" height="56" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="405" y="343" text-anchor="middle" font-weight="bold" fill="#334155">direction == DOWN ?</text>
    <text x="405" y="358" text-anchor="middle" font-weight="bold" fill="#334155">serveDown() : serveUp()</text>
    <rect x="560" y="320" width="210" height="56" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="665" y="343" text-anchor="middle" font-weight="bold" fill="#334155">sweep empty → reverse;</text>
    <text x="665" y="358" text-anchor="middle" font-weight="bold" fill="#334155">both empty → IDLE</text>
  </g>

</svg>
```

The direction flip is worth calling out. When the elevator finishes its up sweep, the down stops remain, and `serveUp` hands control back to `run`, which now sees `direction == DOWN` and serves the down set. A request that arrived while moving up for a floor below (a cabin request pressed "too late") is a known quirk of SCAN: it waits for the return trip. If the interviewer asks about that, the honest answer is that this is exactly the trade-off SCAN makes, predictability over latency for out-of-order requests.

## Design Patterns Used

The State pattern fits the elevator's moving parts, but I want to be precise about why. The elevator has three states, idle, moving up, and moving down, and the scheduling behavior genuinely differs by state. A state machine expressed as an enum with a `nextAction()` method, or a small set of state classes, is a legitimate use. What is not legitimate is wiring Observer everywhere so the floor display "reacts" to elevator movement. The display can poll the controller or the controller can push to it; neither needs an event bus in a system this small. And no Command pattern wrapping every button press. A `Request` object is already a command; wrapping it again is ceremony. One pattern, honestly placed, beats four pattern-shaped artifacts.

## Handling Edge Cases / Concurrency

The interesting edges are scheduling, not thread safety, but there is one concurrency story worth naming. In a real building, floor buttons are pressed by many people, so the controller must be safe against concurrent `addRequest` calls. The two `TreeSet`s are the shared mutable state, and the fix is a single lock around mutation and the run loop, or draining requests atomically at the start of each sweep. In the interview, naming that race and its fix is more valuable than writing a threaded simulation.

The scheduling edges to reason through: a request for the floor you are already on should be dropped (handled by the early return), a stale request for a floor you have passed should be ignored rather than causing a reversal, and the "same floor pressed twice" case should not double-open the door. Each of these is a two-line guard. Candidates who know to look for them are candidates who have run the walkthrough, not just drawn the classes.

## Common Mistakes

The number one failure is modeling the elevator as a loop over a flat list of floors, checking "does anyone want floor 3?" every time it passes. That is a passenger elevator with the intelligence of a freight lift. The direction-aware sets are not a refinement, they are the algorithm.

The second mistake is deciding direction in `Elevator`. The elevator should never look at `upStops`; the moment it does, the controller has lost its job and the follow-up question about a second elevator turns into surgery.

The third mistake is ignoring the direction semantics of floor calls. A person on floor 5 pressing the down arrow means "take me down." Serving that call while the cabin is on an up-sweep and opening doors is wrong on two counts: the passenger wants down, and the cabin would open for a ride going the wrong way. The `Request.direction` field exists precisely to prevent this, and designs that drop it fail the walkthrough instantly.

## Interview Perspective

A weak answer produces an `Elevator` class with a `moveUp()` and `moveDown()` method and a `goToFloor(int)` that just sets a field, then nothing else. When the interviewer asks "a passenger on floor 4 wants to go up, and the elevator is on floor 2 heading down, what happens?" there is no answer, because there is no decision-making anywhere in the design.

A strong answer points at the controller and says "the elevator is on floor 2 heading down with a down-stop at floor 1; the floor-4 up call goes into the up set; the controller finishes the down sweep, reverses, and serves floor 4 on the way up." That answer names the algorithm, the data structures, and the sequence in one breath. The follow-up twists are predictable and should land cleanly: "what if two passengers press opposite directions on the same floor" (the cabin stops once, direction is chosen by the first call served), "what if there are two elevators" (the controller owns a list, and dispatch picks the nearest idle one or the one heading the same way, which is where the strategy seam goes), "what if a cabin request is for a floor already passed" (SCAN quirk, it waits for the sweep). Strong candidates volunteer the SCAN trade-off before being asked.

## Knowledge Check

1. The elevator is on floor 3, idle. A cabin request arrives for floor 7, then a floor call arrives at floor 2 pressing UP. Trace the controller's state after each event and the full sequence of stops.
2. Why does putting both up and down requests in one unsorted list break the system? Describe the exact wrong behavior it produces with a concrete example.
3. You add a second elevator. The current design has a single controller owning one elevator. What is the minimal change, and where would a dispatch policy like "serve from the nearest elevator" plug in?

## Key Takeaways

- The elevator is a dumb machine and the controller is the brain. Never merge them.
- Two direction-keyed sorted sets are the scheduling algorithm. The data structure does the thinking.
- Floor calls carry a direction; honoring it is the difference between a working system and a passenger-elevator joke.
- SCAN is a trade-off: predictable sweeps, late requests wait for the return trip. Say so before you are asked.
- Concurrency here is one lock around the request sets and the run loop. Name it, fix it, move on.

## What's Next

The elevator forced you to build a controller with an internal decision procedure. The vending machine keeps the "input, decide, output" shape but moves the decision to a state machine, and it introduces money, which is its own kind of trap.

---

This article explains how to design an elevator system by separating the dumb machine from the controller that decides the next stop. Its point of view is that the two direction-keyed sorted sets are the algorithm, and scheduling in the elevator turns follow-ups into rewrites.
