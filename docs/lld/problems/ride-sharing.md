# Design a Ride-Sharing App (Uber, Simplified)

> **Time:** 60+ minutes

## Problem

Design the object model behind a simplified Uber. The system must:

- Let **riders** request a ride from a pickup location to a destination.
- Let **drivers** mark themselves available and accept ride requests.
- **Match** a rider to a nearby available driver.
- Track the **ride lifecycle**: requested → driver matched → en route to pickup → in trip → completed.
- Compute the **fare** based on distance, duration, and surge.
- Let riders and drivers **rate** each other after the trip.

## Real-world intuition

Uber's LLD is built on three core abstractions: a **driver pool** (geo-indexed for fast nearest-neighbor lookup), a **matching strategy** (which driver gets which ride), and a **ride state machine**. The interesting design pressures are: matching must be pluggable (UberX vs UberPool vs Premier all match differently); fare must be pluggable (surge, regional, promotional); and ride state must be tightly controlled so the same ride can't both be cancelled and completed.

## Key classes

- `Rider` — profile, current location, ride history, payment method.
- `Driver` — profile, vehicle, current location, availability flag, rating.
- `Vehicle` — type (Standard, XL, Black), license plate, capacity.
- `Location` — lat/long, with distance utility.
- `Ride` — links a `Rider`, a `Driver`, pickup, dropoff, status, fare.
- `RideMatchingService` — finds the best driver for a ride request.
- `MatchingStrategy` (interface) — `NearestDriver`, `HighestRated`, `Pool` (shared rides).
- `FareCalculator` — computes fare given route, time, surge.
- `FareStrategy` (interface) — per ride-type fare logic.
- `Rating` — score + comment, attached to a `Ride`.

## Design patterns used

- **Strategy (MatchingStrategy, FareStrategy)** — both axes have multiple valid implementations and change frequently. Strategy is the right pattern for both. Adding UberPool to the matching is one new class.
- **State (Ride status)** — `REQUESTED` → `MATCHED` → `EN_ROUTE_TO_PICKUP` → `IN_TRIP` → `COMPLETED` (or `CANCELLED` from various points). State pattern keeps each phase's allowed actions explicit.
- **Observer (RideObserver)** — push notifications, ETA updates, surge alerts, driver-arrival alerts. All fan-out targets subscribe to ride state changes.
- **Singleton (RideMatchingService)** — one matching service per region/process; legitimate global.

## Class diagram (sketch)

```text
   ┌────────┐   matched to   ┌────────┐
   │ Rider  │◇──────────────│ Driver │◇──▶ Vehicle
   └───┬────┘                └───┬────┘
       │ requests                │ accepts
       ▼                         ▼
              ┌────────┐
              │  Ride  │  «state-driven: REQUESTED → COMPLETED»
              ├────────┤
              │ pickup │
              │ dropoff│
              │ status │
              │ fare   │
              └────────┘

   RideMatchingService ──▶ MatchingStrategy «interface»
                          △
                          ┊
              NearestDriver  Pool  HighestRated

   FareCalculator ──▶ FareStrategy «interface»
                       △
                       ┊
           Standard  Surge  Premier  PoolFare
```

## Code sketch

```java
enum RideStatus { REQUESTED, MATCHED, EN_ROUTE_TO_PICKUP, IN_TRIP, COMPLETED, CANCELLED }

class Location {
    final double lat, lng;
    Location(double lat, double lng) { this.lat = lat; this.lng = lng; }
    public double distanceKm(Location other) { /* haversine */ return 0; }
}

class Driver {
    private final String id;
    private Vehicle vehicle;
    private Location location;
    private boolean available = true;
    private double rating = 5.0;
}

interface MatchingStrategy {
    Optional<Driver> match(Rider rider, Location pickup, List<Driver> available);
}

class NearestDriverStrategy implements MatchingStrategy {
    private static final double MAX_RADIUS_KM = 5.0;

    public Optional<Driver> match(Rider rider, Location pickup, List<Driver> available) {
        return available.stream()
            .filter(d -> d.location().distanceKm(pickup) <= MAX_RADIUS_KM)
            .min(Comparator.comparingDouble(d -> d.location().distanceKm(pickup)));
    }
}

interface FareStrategy {
    double calculate(Route route, Instant when, double surgeMultiplier);
}

class StandardFare implements FareStrategy {
    private static final double BASE = 50;
    private static final double PER_KM = 12;
    private static final double PER_MIN = 1.5;

    public double calculate(Route r, Instant when, double surge) {
        double cost = BASE + r.distanceKm() * PER_KM + r.durationMin() * PER_MIN;
        return cost * surge;
    }
}

class Ride {
    private final Rider rider;
    private Driver driver;
    private final Location pickup, dropoff;
    private RideStatus status = RideStatus.REQUESTED;
    private double fare;
    private final List<RideObserver> observers;

    public void assignDriver(Driver d) {
        if (status != RideStatus.REQUESTED) throw new IllegalStateException();
        this.driver = d;
        transitionTo(RideStatus.MATCHED);
    }

    public void startPickup()    { transitionTo(RideStatus.EN_ROUTE_TO_PICKUP); }
    public void startTrip()      { transitionTo(RideStatus.IN_TRIP); }
    public void complete(double fare) {
        this.fare = fare;
        transitionTo(RideStatus.COMPLETED);
    }
    public void cancel() {
        if (status == RideStatus.COMPLETED) throw new IllegalStateException();
        transitionTo(RideStatus.CANCELLED);
    }

    private void transitionTo(RideStatus next) {
        this.status = next;
        observers.forEach(o -> o.onStatusChange(this));
    }
}

class RideMatchingService {
    private static final RideMatchingService INSTANCE = new RideMatchingService();
    private final List<Driver> drivers = new ArrayList<>();
    private MatchingStrategy strategy = new NearestDriverStrategy();
    private final FareCalculator fareCalc = new FareCalculator(new StandardFare());

    public Ride requestRide(Rider rider, Location pickup, Location dropoff) {
        Ride ride = new Ride(rider, pickup, dropoff);
        List<Driver> available = drivers.stream().filter(Driver::isAvailable).toList();
        Driver d = strategy.match(rider, pickup, available)
                          .orElseThrow(NoDriversAvailableException::new);
        d.setAvailable(false);
        ride.assignDriver(d);
        return ride;
    }
}
```

## Key design decisions

1. **Matching and fare are both Strategies** — they're the two most volatile axes. Surge pricing is a tweak to `FareStrategy`; UberPool is a tweak to `MatchingStrategy`. Both must be swappable per ride type without disturbing the rest of the system.
2. **Ride is the state container** — driver, rider, fare, status, and observers all live on the `Ride`. The matching service owns the driver pool but not the rides.
3. **Explicit state transitions with guards** — `assignDriver` only works from `REQUESTED`; `complete` only from `IN_TRIP`; `cancel` rejects from `COMPLETED`. Invalid transitions throw immediately, not silently corrupt state.
4. **Drivers leave the pool on match** — `setAvailable(false)` prevents the same driver from being matched to two rides. Critical race-condition guard.
5. **Distance uses a geo utility (haversine)** — encapsulated in `Location.distanceKm`. For real systems, swap in a geohash-indexed pool for fast nearest-neighbor; the interface stays the same.
6. **Rating is a separate object attached to the completed ride** — it's not on `Driver` or `Rider` directly; it's *between* them, captured per trip. Aggregating ratings into a profile is a downstream computation.
7. **Cancellation policy is on `Ride`, not on the matching service** — different ride states have different cancel rules (free before match, fee after pickup en-route). Keeping the rule in `Ride.cancel()` keeps it close to the state.

## State diagram

```text
   ┌──────────┐ match  ┌────────┐ pickup ┌──────────────────┐ start ┌─────────┐ complete ┌──────────┐
   │ REQUESTED│───────▶│MATCHED │───────▶│EN_ROUTE_TO_PICKUP│──────▶│ IN_TRIP │─────────▶│COMPLETED │
   └────┬─────┘        └───┬────┘        └────────┬─────────┘       └────┬────┘          └──────────┘
        │                  │                      │                      │
        │ cancel           │ cancel               │ cancel               │ (cannot cancel from IN_TRIP without trip-end logic)
        ▼                  ▼                      ▼                      ▼
                              CANCELLED
```

## Extensibility

**Add UberPool (shared rides):**

1. `class PoolMatchingStrategy implements MatchingStrategy` — matches a rider with an in-progress ride going in a similar direction.
2. `class PoolFare implements FareStrategy` — discounted fare, possibly split across riders.
3. `Ride` may need to support multiple riders.

**Add surge pricing:**

1. `SurgeAwareFare` wraps the base fare strategy, applying a per-region multiplier.
2. The matching service reads region demand from a separate `SurgeService`.

**Add scheduled rides:**

1. New `ScheduledRide` enters a different state machine (`SCHEDULED` → `DISPATCHING` → `MATCHED`...).
2. A background job promotes scheduled rides to matched at the right time.

**Add a different vehicle class (UberBlack):**

1. `Vehicle.type` already exists.
2. `MatchingStrategy` filters by vehicle type; `FareStrategy` knows premium pricing.
3. No structural changes.

**Add driver heatmap (where demand is):**

1. `DemandHeatmapObserver` subscribes to `RideRequested` events; aggregates to a region-level view.
2. Drivers consult it independently of the matching engine.

## Linked concepts

- [OOP](../fundamentals/oop.md) — composition of rider/driver/ride/vehicle.
- [SOLID](../fundamentals/solid.md) — OCP via matching and fare strategies; SRP across matching service, ride state, fare calculator.
- [Design Patterns](../fundamentals/design-patterns.md) — Strategy (matching, fare), State (ride status), Observer (notifications), Singleton (matching service).
