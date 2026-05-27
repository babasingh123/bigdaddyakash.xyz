# Design a Car Rental System

> **Time:** 40–50 minutes

## Problem

Design the software behind a car rental service (Hertz, Zipcar). The system must:

- Support multiple **car types** (Economy, Sedan, SUV, Luxury).
- Let customers **search** available cars by location, date range, and type.
- **Book** a car for a date range; **return** it when done.
- Compute the **rental fee** (base + days, possibly with type and seasonal multipliers).
- Compute **late fees** if the car is returned past the due date.
- Manage cars across multiple **locations** (pick up here, drop off there).

## Real-world intuition

Car rental is a hotel-booking variant: a finite, time-sliced inventory with a pricing strategy. The interesting twist is **multi-location** — the car you rent in San Francisco may end up returned in Oakland, and the system has to track that the car is now physically there. That logistics layer is the LLD wrinkle.

## Key classes

- `CarRentalSystem` — top-level orchestrator; owns locations and cars.
- `Car` (abstract) — registration, model, type, daily rate, current location, status (`AVAILABLE`, `RENTED`, `MAINTENANCE`).
- `EconomyCar`, `SedanCar`, `SuvCar`, `LuxuryCar` — concrete subclasses, primarily differing in daily rate and amenities.
- `Location` — branch/pickup point; owns the cars currently parked there.
- `Customer` — driver license, address, rental history.
- `Booking` — links a `Customer`, a `Car`, pickup/dropoff locations, date range, computed fees, status.
- `FeeCalculator` — base rental fee.
- `LateFeeCalculator` — overdue fee, separate concern.
- `RentalStrategy` (interface) — base fee algorithm: `FlatDailyRate`, `WeekendSurcharge`, `LongRentalDiscount`.

## Design patterns used

- **Strategy (RentalStrategy, LateFeeStrategy)** — pricing rules change seasonally, by promotion, and per customer tier. Strategy pattern keeps them swappable.
- **Factory (CarFactory)** — creating cars from configuration (model, type code) is cleanly delegated.
- **Singleton (CarRentalSystem)** — one system per deployment; legitimate global.
- **State (Car status, Booking status)** — explicit transitions: car AVAILABLE → RENTED → AVAILABLE; booking RESERVED → PICKED_UP → RETURNED.

## Class diagram (sketch)

```text
   ┌──────────────────┐
   │ CarRentalSystem  │ «Singleton»
   ├──────────────────┤
   │ -locations       │◇──▶ Location ◇──▶ Car (currently parked here)
   │ -bookings        │◇──▶ Booking
   │ -rentalStrategy  │──▶ RentalStrategy «interface»
   │ -lateFeeStrategy │──▶ LateFeeStrategy «interface»
   └──────────────────┘

   ┌──────────┐  «abstract»
   │   Car    │
   ├──────────┤
   │ -reg     │
   │ -type    │
   │ -rate    │
   │ -location│
   │ -status  │
   └────△─────┘
   ┌────┼─────┬─────┬──────┐
 Economy  Sedan  SUV  Luxury
```

## Code sketch

```java
enum CarStatus { AVAILABLE, RENTED, MAINTENANCE }
enum BookingStatus { RESERVED, PICKED_UP, RETURNED, CANCELLED }

abstract class Car {
    protected final String registration;
    protected final String model;
    protected double dailyRate;
    protected Location currentLocation;
    protected CarStatus status = CarStatus.AVAILABLE;

    public abstract CarType type();
}

class SedanCar extends Car {
    public CarType type() { return CarType.SEDAN; }
}

class Location {
    private final String id;
    private final String address;
    private final List<Car> cars = new ArrayList<>();
}

class Booking {
    private final Customer customer;
    private final Car car;
    private final Location pickupLocation;
    private final Location dropoffLocation;
    private final LocalDate pickupDate;
    private final LocalDate dueDate;
    private LocalDate actualReturnDate;
    private double rentalFee;
    private double lateFee;
    private BookingStatus status = BookingStatus.RESERVED;
}

interface RentalStrategy {
    double fee(Car car, LocalDate from, LocalDate to);
}

class FlatDailyRate implements RentalStrategy {
    public double fee(Car car, LocalDate from, LocalDate to) {
        long days = ChronoUnit.DAYS.between(from, to);
        return car.dailyRate() * days;
    }
}

class LongRentalDiscount implements RentalStrategy {
    private final RentalStrategy inner;
    public LongRentalDiscount(RentalStrategy i) { this.inner = i; }
    public double fee(Car c, LocalDate from, LocalDate to) {
        double base = inner.fee(c, from, to);
        long days = ChronoUnit.DAYS.between(from, to);
        return days >= 7 ? base * 0.9 : base;
    }
}

interface LateFeeStrategy {
    double lateFee(Booking b, LocalDate returnedOn);
}

class FlatLateFee implements LateFeeStrategy {
    public double lateFee(Booking b, LocalDate returnedOn) {
        long lateDays = Math.max(0, ChronoUnit.DAYS.between(b.dueDate(), returnedOn));
        return lateDays * b.car().dailyRate() * 1.5;       // 1.5× normal day rate
    }
}

class CarRentalSystem {
    private static final CarRentalSystem INSTANCE = new CarRentalSystem();
    private final Map<String, Location> locations = new HashMap<>();
    private RentalStrategy rentalStrategy = new FlatDailyRate();
    private LateFeeStrategy lateFeeStrategy = new FlatLateFee();

    public List<Car> search(CarType type, Location pickup, LocalDate from, LocalDate to) {
        return pickup.cars().stream()
            .filter(c -> c.type() == type)
            .filter(c -> isAvailable(c, from, to))
            .toList();
    }

    public Booking book(Customer c, Car car, Location pickup, Location dropoff, LocalDate from, LocalDate to) {
        if (!isAvailable(car, from, to)) throw new CarUnavailableException();
        double fee = rentalStrategy.fee(car, from, to);
        Booking b = new Booking(c, car, pickup, dropoff, from, to, fee);
        car.setStatus(CarStatus.RENTED);                   // reservation holds the car
        return b;
    }

    public void returnCar(Booking b, LocalDate on) {
        b.markReturned(on);
        b.setLateFee(lateFeeStrategy.lateFee(b, on));
        b.car().setStatus(CarStatus.AVAILABLE);
        b.car().setLocation(b.dropoffLocation());          // car is now at the drop-off branch
    }

    private boolean isAvailable(Car c, LocalDate from, LocalDate to) { /* check overlapping bookings */ return false; }
}
```

## Key design decisions

1. **Pickup vs dropoff locations** — they can differ. The car *moves* to the drop-off branch on return. This is the meaningful logistics piece that distinguishes car rental from hotel rooms.
2. **Two separate strategies: rental and late fee** — they're independent concerns. Rental fee depends on cars and base policy; late fee depends on overdue policy. Don't conflate them.
3. **Strategy + decorator composition** — `LongRentalDiscount` wraps another `RentalStrategy`. This is how seasonal/promotional adjustments stack without combinatorial classes.
4. **Car status reflects physical state** — `AVAILABLE`, `RENTED`, `MAINTENANCE`. Booking status reflects the rental lifecycle. They are related but not the same — a car may be `AVAILABLE` while a booking is `RESERVED` (future date).
5. **Abstract `Car` with type subclasses** — each type has a different default rate and amenity set. New types (electric, convertible) plug in cleanly.
6. **`CarFactory` for creation** — taking a config row and producing the right `Car` subclass. Centralized so adding a type doesn't touch every code path.

## Extensibility

**Add electric / EV cars with charging considerations:**

1. `class ElectricCar extends Car` with a `range` and `chargeLevel`.
2. Return flow checks charge level; charges a fee if below threshold.
3. Existing rental flow unchanged.

**Add insurance options:**

1. `Insurance` add-on associated with a booking, with its own price strategy.
2. `Booking.totalFee = rentalFee + insurance + lateFee`.

**Add per-mile pricing:**

1. New `RentalStrategy` variant: `PerMileRate` reads odometer at return.
2. Inject when the customer chooses that plan.

**Add corporate accounts (volume discounts):**

1. `Customer` gains a `tier` or `corporateAccount` reference.
2. Strategy reads it and applies discount; or wrap with `CorporateDiscountStrategy`.

**Add multiple-day reservations and a fleet manager view:**

1. Already supported — bookings are time-sliced.
2. New view aggregates per-car utilization for management dashboard.

## Linked concepts

- [OOP](../fundamentals/oop.md) — abstract `Car` with type subclasses; composition of system → locations → cars.
- [SOLID](../fundamentals/solid.md) — SRP (rental fee vs late fee), OCP (new car types and strategies).
- [Design Patterns](../fundamentals/design-patterns.md) — Strategy, Factory, Singleton; light decorator for stacking pricing rules.
