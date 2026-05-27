# Design an Elevator System

> **Time:** 45–60 minutes

## Problem

Design the software controlling an elevator bank in a multi-story building. The system must:

- Manage **multiple elevators** in the same building.
- Accept ride **requests** from any floor (hall buttons) and any elevator (cabin buttons).
- **Optimize** total waiting time across requests (don't just FCFS).
- Handle **direction** correctly — an elevator going up should serve all up-requests on its path before reversing.
- Support **states**: `MOVING`, `STOPPED`, `MAINTENANCE`, `EMERGENCY`.
- Support an **emergency stop** that overrides everything.

## Real-world intuition

This problem is interesting because the *system* (the controller) and the *device* (each elevator) have very different concerns. The controller solves dispatch — *which elevator should answer this request?* — while each elevator solves motion — *given my queue, what's my next stop?* Separating them cleanly is the design call interviewers are looking for.

It's also the textbook example of the **State pattern** — an elevator's behavior on every input (button press, sensor reading) depends on its current state. A `switch (currentState)` mess across 10 methods is the smell that State eliminates.

## Key classes

- `ElevatorController` — singleton; receives all hall requests; dispatches the best elevator.
- `Elevator` — one physical car; owns its current floor, direction, state, and pending stops.
- `ElevatorState` (interface or enum-driven) — defines behavior in each state: `Idle`, `Moving`, `Stopped`, `Maintenance`, `Emergency`.
- `Direction` (enum) — `UP`, `DOWN`, `IDLE`.
- `Request` — a ride request: source floor, destination floor (for cabin button), direction (for hall button), timestamp.
- `Button` (abstract) → `HallButton` (per floor, direction-specific) and `ElevatorButton` (per cabin, floor-specific).
- `DispatchStrategy` (interface) — algorithm for picking the best elevator: `FCFS`, `Scan`, `Look`, `NearestCar`.

## Design patterns used

- **State (Elevator behavior)** — by far the most important pattern here. An elevator behaves completely differently when it's `Moving` vs `Stopped` vs in `Maintenance`. Encoding state as objects with their own `onArrive`, `onRequest`, `onEmergency` methods removes a thicket of conditionals.
- **Strategy (DispatchStrategy)** — picking the best elevator for a request is an algorithm with multiple valid implementations (FCFS, SCAN, LOOK, nearest car). Strategy lets the building owner tune for speed vs energy without touching the controller.
- **Singleton (ElevatorController)** — there's exactly one dispatcher per building. It coordinates all elevators, so a stable global reference is genuinely useful.
- **Observer (status displays)** — floor displays subscribe to elevator state changes to show "elevator 3 is on floor 7, going up". Decouples the UI from the elevator internals.

## Class diagram (sketch)

```text
              ┌─────────────────────┐
              │ ElevatorController  │ «Singleton»
              ├─────────────────────┤
              │ -elevators          │
              │ -dispatchStrategy   │
              │ +request(r)         │
              └──────────┬──────────┘
                         ◆ 1..*
                  ┌──────▼──────┐
                  │  Elevator   │
                  ├─────────────┤
                  │ -currentFloor│
                  │ -direction   │
                  │ -state       │  ─────▶  ElevatorState «interface»
                  │ -queue       │                  △
                  │ +addStop(f)  │                  ┊
                  └─────────────┘   Idle  Moving  Stopped  Maintenance  Emergency

         DispatchStrategy «interface»
                △
                ┊
        FCFS    LOOK    NearestCar
```

## Code sketch

```java
interface ElevatorState {
    void onRequest(Elevator e, Request r);
    void onArriveAtFloor(Elevator e, int floor);
    void onEmergencyStop(Elevator e);
}

class IdleState implements ElevatorState {
    public void onRequest(Elevator e, Request r) {
        e.addStop(r.getSource());
        e.setState(new MovingState());
    }
    public void onArriveAtFloor(Elevator e, int floor) { /* unreachable */ }
    public void onEmergencyStop(Elevator e) { e.setState(new EmergencyState()); }
}

class MovingState implements ElevatorState {
    public void onRequest(Elevator e, Request r) {
        e.addStop(r.getSource());        // queue insertion respects direction
    }
    public void onArriveAtFloor(Elevator e, int floor) {
        e.removeStop(floor);
        e.openDoors();
        e.setState(new StoppedState());
    }
    public void onEmergencyStop(Elevator e) { e.haltImmediately(); e.setState(new EmergencyState()); }
}

class Elevator {
    private int currentFloor = 0;
    private Direction direction = Direction.IDLE;
    private ElevatorState state = new IdleState();
    private final TreeSet<Integer> upStops = new TreeSet<>();
    private final TreeSet<Integer> downStops = new TreeSet<>(Comparator.reverseOrder());

    public void request(Request r)            { state.onRequest(this, r); }
    public void onArrive(int floor)           { state.onArriveAtFloor(this, floor); }
    public void emergencyStop()               { state.onEmergencyStop(this); }
    void setState(ElevatorState s)            { this.state = s; }

    int nextStop() {
        return direction == Direction.UP ? upStops.first() : downStops.first();
    }
}

interface DispatchStrategy {
    Elevator pickElevator(List<Elevator> elevators, Request r);
}

class LookStrategy implements DispatchStrategy {
    public Elevator pickElevator(List<Elevator> elevators, Request r) {
        // Prefer an elevator already moving toward r.getSource() in r's direction;
        // otherwise the nearest idle one.
        return elevators.stream()
            .min(Comparator.comparingInt(e -> cost(e, r)))
            .orElseThrow();
    }
    private int cost(Elevator e, Request r) { /* ... */ return 0; }
}

class ElevatorController {
    private static final ElevatorController INSTANCE = new ElevatorController();
    private final List<Elevator> elevators = new ArrayList<>();
    private DispatchStrategy strategy = new LookStrategy();

    public void onHallButton(Request r) {
        Elevator chosen = strategy.pickElevator(elevators, r);
        chosen.request(r);
    }
}
```

## Key design decisions

1. **State pattern over `switch`** — every elevator method (`onRequest`, `onArrive`, `emergencyStop`) routes through the state object. Adding a new state (e.g., `EnergySavingState` that won't move below floor 5 at night) is a new class, no edits to existing ones.
2. **Controller owns dispatch, elevator owns motion** — clean SRP split. The controller knows about the bank; each elevator only knows about itself. Strategy can change without elevators noticing.
3. **Two ordered sets for stops (`upStops`, `downStops`)** — captures the SCAN/LOOK algorithm naturally. Going up, serve `upStops` in ascending order; flip to `downStops` when up is empty.
4. **Hall vs cabin requests are different inputs** — hall requests carry a *direction*; cabin requests carry a *destination*. They share `Request` as a base but differ in dispatch handling.
5. **Emergency is a state, not a flag** — once in `EmergencyState`, every input is rejected uniformly until reset. A boolean would be checked in every method; the state object replaces all those checks.
6. **Singleton controller, injected strategy** — exactly one bank controller per building; strategy is constructor- or setter-injected so building owners can tune behavior.

## Dispatch algorithms (quick reference)

| Algorithm | Strategy | Trade-off |
|-----------|---------|-----------|
| **FCFS** | Serve requests in arrival order. | Simple, but very inefficient — lots of wasted travel. |
| **SCAN** (Elevator Algorithm) | Move in one direction, serve everything in that direction, then reverse. Like a window-washer. | Fair, predictable, but visits ends even when no requests live there. |
| **LOOK** | Like SCAN, but reverse at the *last actual request* rather than the building's end. | Strictly faster than SCAN; same fairness. Most real systems use this or a variant. |
| **Nearest Car** | Assign request to whichever elevator can reach it soonest. | Lower average wait, but can starve far-away floors under load. |

## Extensibility

**Add a new state (Maintenance):**

1. Create `class MaintenanceState implements ElevatorState` — all methods reject requests, allow only manual movement commands.
2. `Elevator.startMaintenance()` flips to it. No edits to other states.

**Add a new dispatch algorithm:**

1. Implement `class GeneticDispatchStrategy implements DispatchStrategy`.
2. Inject it via `controller.setStrategy(...)`. Elevators unaware.

**Add VIP priority:**

1. `Request` gains a `priority` field.
2. The dispatch strategy considers priority in its cost function.
3. Existing strategies still work because they ignore the field.

**Add multi-zone elevators (some serve only floors 1–10):**

1. Each `Elevator` gets a `Set<Integer> serviceableFloors`.
2. The dispatch strategy filters out elevators that can't serve the requested floor.

## Linked concepts

- [OOP](../fundamentals/oop.md) — State as polymorphism over a status field.
- [SOLID](../fundamentals/solid.md) — SRP (controller vs elevator), OCP (new states/strategies), DIP (inject strategy).
- [Design Patterns](../fundamentals/design-patterns.md) — State, Strategy, Singleton, Observer.
