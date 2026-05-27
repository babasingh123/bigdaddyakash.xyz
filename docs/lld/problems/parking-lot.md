# Design a Parking Lot

> **Time:** 30–45 minutes

## Problem

Design the software for a multi-floor parking lot. The system must:

- Support multiple **floors**, each with multiple **spot types**: Compact, Large, Handicapped, Motorcycle.
- Support multiple **vehicle types**: Car, Truck, Motorcycle, Van — each compatible with specific spot types.
- Track which spots are **available** vs **occupied** in real time.
- Issue a **parking ticket** on entry capturing the entry time.
- Calculate the **fee** on exit based on duration (and possibly vehicle/spot type).
- Support multiple **payment methods** (cash, card, UPI).

## Real-world intuition

This is the canonical "design a system with hierarchies and strategies" problem. Real-world airport and mall parking systems are exactly this — and they all face the same design pressures: new vehicle types appear (electric scooters), new spot types appear (EV charging), and pricing changes seasonally. A clean LLD shows how to absorb those changes without rewriting the core.

## Key classes

- `ParkingLot` — top-level singleton; owns floors; entry point for `parkVehicle` / `unparkVehicle`.
- `ParkingFloor` — one level of the lot; owns its spots; tracks availability per spot type.
- `ParkingSpot` (abstract) — base class for spots; knows its size class and whether it's free.
    - `CompactSpot`, `LargeSpot`, `HandicappedSpot`, `MotorcycleSpot` — concrete subclasses.
- `Vehicle` (abstract) — base for all vehicles; carries license plate and required spot type.
    - `Car`, `Truck`, `Motorcycle`, `Van` — concrete subclasses.
- `ParkingTicket` — issued on entry; records spot, vehicle, entry time; consumed at exit.
- `PaymentStrategy` (interface) — `CashPayment`, `CardPayment`, `UpiPayment`.
- `FeeCalculator` — given a ticket, computes the amount due.
- `EntryGate` / `ExitGate` — coordinate ticket issuance, payment, and spot release.

## Design patterns used

- **Singleton (ParkingLot)** — there is exactly one parking lot per deployment. A singleton gives global access to the lot's state from any entry/exit gate without threading the reference through every method. Use sparingly; this is one of the cases where it genuinely fits.
- **Strategy (PaymentStrategy, FeeCalculator)** — payment methods and fee rules vary independently and are likely to grow (add Apple Pay, add weekend pricing). Encapsulating each as a class lets us swap behavior at runtime and add new strategies without touching the lot.
- **Factory (VehicleFactory)** — vehicle creation may depend on input data (license plate, type code from a barcode scan). A factory centralizes the mapping from input → concrete `Vehicle` subclass, keeping the gate code clean.
- **Observer (availability updates)** — display boards at the entrance want to be notified when a spot type becomes available or full. The lot publishes events; subscribers (displays, mobile apps) react.
- **Abstract classes (Vehicle, ParkingSpot)** — both have shared state (id, type) and a polymorphic shape. Abstract bases let us extend with new types without touching the lot's core logic (OCP).

## Class diagram (sketch)

```text
                ┌──────────────┐
                │  ParkingLot  │ «Singleton»
                ├──────────────┤
                │ -floors      │
                │ +park(v)     │
                │ +unpark(t)   │
                └──────┬───────┘
                       ◆ 1..*
                ┌──────▼──────┐
                │ ParkingFloor│
                └──────┬──────┘
                       ◆ 1..*
                ┌──────▼──────┐
                │ ParkingSpot │ «abstract»
                ├─────────────┤
                │ -id, -free  │
                │ +fits(v)    │
                └──────△──────┘
        ┌──────────────┼──────────────┬─────────────┐
   CompactSpot     LargeSpot    HandicappedSpot  MotorcycleSpot

                ┌──────────┐
                │ Vehicle  │ «abstract»
                └────△─────┘
        ┌───────────┼────────────┬─────────┐
       Car        Truck      Motorcycle    Van

ParkingLot ──▶ PaymentStrategy «interface»
                   △
                   ┊
        CashPayment   CardPayment   UpiPayment
```

## Code sketch

```java
// Abstract base — every spot type shares id, occupancy, and a compatibility check.
abstract class ParkingSpot {
    protected final String id;
    protected boolean free = true;
    protected Vehicle vehicle;

    public abstract boolean fits(Vehicle v);

    public synchronized boolean assign(Vehicle v) {
        if (!free || !fits(v)) return false;
        this.vehicle = v;
        this.free = false;
        return true;
    }

    public synchronized void release() {
        this.vehicle = null;
        this.free = true;
    }
}

class CompactSpot extends ParkingSpot {
    public boolean fits(Vehicle v) {
        return v.getType() == VehicleType.CAR || v.getType() == VehicleType.MOTORCYCLE;
    }
}

// Strategy interface keeps the lot decoupled from payment provider details.
interface PaymentStrategy {
    boolean pay(double amount);
}

class CardPayment implements PaymentStrategy {
    public boolean pay(double amount) { /* call gateway */ return true; }
}

// The Singleton — one lot, global access. Owns floors via composition.
class ParkingLot {
    private static final ParkingLot INSTANCE = new ParkingLot();
    private final List<ParkingFloor> floors = new ArrayList<>();
    private final FeeCalculator feeCalculator = new HourlyFeeCalculator();

    private ParkingLot() {}
    public static ParkingLot get() { return INSTANCE; }

    public ParkingTicket park(Vehicle v) {
        for (ParkingFloor f : floors) {
            ParkingSpot s = f.findSpotFor(v);
            if (s != null && s.assign(v)) {
                return new ParkingTicket(v, s, Instant.now());
            }
        }
        throw new NoSpotAvailableException();
    }

    public double unpark(ParkingTicket t, PaymentStrategy p) {
        double fee = feeCalculator.calculate(t);
        if (!p.pay(fee)) throw new PaymentFailedException();
        t.getSpot().release();
        return fee;
    }
}
```

## Key design decisions

1. **Singleton for `ParkingLot`** — only one logical parking system per deployment. Avoids passing the lot reference through every gate, sensor, and display. Acceptable here because the lot really is global.
2. **Strategy for `PaymentStrategy` and `FeeCalculator`** — payment providers and pricing rules change frequently and independently. Both are obvious Strategy candidates; both are injected so they can be swapped or mocked.
3. **Abstract `Vehicle` and `ParkingSpot`** — new vehicle types and spot types will be added. Abstract bases plus polymorphism let us extend without modifying the lot's existing code (OCP).
4. **Composition over inheritance** — `ParkingLot` HAS-A floors, `ParkingFloor` HAS-A spots. Nothing inherits from "Lot" or "Floor"; the hierarchy is only in vehicle and spot types where IS-A genuinely holds.
5. **Synchronized `assign`/`release`** — multiple gates may try to assign the same spot concurrently. Lock at the spot level (fine-grained) to keep contention low.
6. **`fits(Vehicle)` lives on the spot, not the vehicle** — the spot owns the compatibility rules. Adding a new spot type doesn't require editing any vehicle class.

## Extensibility

**Add a new vehicle type (e.g., ElectricScooter):**

1. Create `class ElectricScooter extends Vehicle` with type `ELECTRIC_SCOOTER`.
2. Update each spot's `fits()` — or, better, drive compatibility from a config table `Map<VehicleType, Set<SpotType>>` so even this step becomes data, not code.
3. No changes to `ParkingLot`, `ParkingFloor`, or payment code.

**Add a new spot type (e.g., EvChargingSpot):**

1. Create `class EvChargingSpot extends ParkingSpot` with its own `fits()`.
2. Register instances on the appropriate floor.
3. The lot's allocation loop just sees one more candidate; no edits.

**Add a new payment method (e.g., ApplePay):**

1. Create `class ApplePayPayment implements PaymentStrategy`.
2. Wire it where the user selects payment.
3. Lot and fee calculator are untouched.

**Add weekend pricing:**

1. Create `class WeekendFeeCalculator implements FeeCalculator`.
2. Decide at runtime which calculator to inject (or wrap them in a `CompositeFeeCalculator` that picks by day).

## Linked concepts

- [OOP](../fundamentals/oop.md) — abstract classes, composition vs inheritance.
- [SOLID](../fundamentals/solid.md) — SRP (one class per concern), OCP (extensible types), DIP (inject payment/fee strategy).
- [Design Patterns](../fundamentals/design-patterns.md) — Singleton, Strategy, Factory, Observer.
