# Design a Hotel Booking System

> **Time:** 40–50 minutes

## Problem

Design the booking system for a hotel chain. The system must:

- Support multiple **room types** (Single, Double, Suite, Deluxe) with different prices and amenities.
- Allow customers to **search** available rooms by **date range** and room type.
- Support **booking** and **cancellation** with appropriate policies.
- Apply a **pricing strategy** that may vary by season, day-of-week, occupancy, or last-minute demand.
- Process **payments** through multiple methods.
- Send **notifications** on booking confirmation, reminders, and cancellations.

## Real-world intuition

Hotel booking sits at the intersection of inventory management and dynamic pricing. Booking.com, Marriott, Airbnb all face the same modeling challenge: rooms are a finite, time-sliced resource. The trick is representing *"is room R available between dates D1 and D2?"* efficiently and atomically — that's the core invariant the design must protect.

## Key classes

- `Hotel` — singleton; owns rooms; entry point for search and booking.
- `Room` (abstract) — base type; has number, type, base price, status; `SingleRoom`, `DoubleRoom`, `Suite`, `DeluxeRoom` extend it.
- `RoomAvailabilityManager` — maintains a date → bookings index per room; answers "available?" queries.
- `Booking` — links a `Customer`, a `Room`, a date range; has status (`CONFIRMED`, `CANCELLED`, `CHECKED_IN`, `CHECKED_OUT`).
- `Customer` — guest profile.
- `PricingStrategy` (interface) — computes the price for a (room, date range, context). Concrete: `FlatPricing`, `SeasonalPricing`, `DynamicPricing`.
- `PaymentStrategy` (interface) — `CardPayment`, `UpiPayment`, `WalletPayment`.
- `NotificationService` — observer of booking events; sends email/SMS/push.

## Design patterns used

- **Singleton (Hotel)** — one hotel per deployment; consistent entry point. (For a chain, you'd parametrize per-hotel; the *system* is still one object.)
- **Factory (RoomFactory)** — creating rooms from configuration (room number, type code, price) is cleanly delegated to a factory.
- **Strategy (PricingStrategy, PaymentStrategy)** — pricing rules and payment methods both vary independently, and both must be swappable per market or per A/B test.
- **Observer (NotificationService)** — multiple parties (customer, hotel staff, housekeeping) react to booking events. Observer decouples them from booking logic.

## Class diagram (sketch)

```text
              ┌──────────┐
              │  Hotel   │ «Singleton»
              ├──────────┤
              │ -rooms   │
              │ +search()│
              │ +book()  │
              └─────┬────┘
                    ◆ 1..*
              ┌─────▼─────┐
              │   Room    │ «abstract»
              └─────△─────┘
        ┌──────┬───┴────┬──────┐
     Single  Double  Suite  Deluxe

   Hotel ──▶ RoomAvailabilityManager
   Hotel ──▶ PricingStrategy «interface»  ─▶  Flat / Seasonal / Dynamic
   Hotel ──▶ Booking ──▶ Customer
                  └──▶ PaymentStrategy
   Hotel ──▶ NotificationService «observer»
```

## Code sketch

```java
enum RoomStatus { AVAILABLE, OCCUPIED, MAINTENANCE }

abstract class Room {
    protected final String number;
    protected final double basePrice;
    public abstract RoomType type();
    public abstract int maxOccupancy();
}

class Suite extends Room {
    public RoomType type() { return RoomType.SUITE; }
    public int maxOccupancy() { return 4; }
}

interface PricingStrategy {
    double price(Room room, LocalDate from, LocalDate to);
}

class SeasonalPricing implements PricingStrategy {
    public double price(Room room, LocalDate from, LocalDate to) {
        double total = 0;
        for (LocalDate d = from; d.isBefore(to); d = d.plusDays(1)) {
            total += room.basePrice() * seasonalMultiplier(d);
        }
        return total;
    }
    private double seasonalMultiplier(LocalDate d) { /* peak vs off-peak */ return 1.0; }
}

class RoomAvailabilityManager {
    // For each room, an ordered set of booked date ranges.
    private final Map<String, NavigableSet<DateRange>> bookings = new HashMap<>();

    public synchronized boolean isAvailable(Room r, DateRange range) {
        NavigableSet<DateRange> existing = bookings.getOrDefault(r.number(), new TreeSet<>());
        return existing.stream().noneMatch(b -> b.overlaps(range));
    }

    public synchronized boolean reserve(Room r, DateRange range) {
        if (!isAvailable(r, range)) return false;
        bookings.computeIfAbsent(r.number(), k -> new TreeSet<>()).add(range);
        return true;
    }
}

class Hotel {
    private static final Hotel INSTANCE = new Hotel();
    private final List<Room> rooms = new ArrayList<>();
    private final RoomAvailabilityManager availability = new RoomAvailabilityManager();
    private PricingStrategy pricing = new SeasonalPricing();
    private final List<BookingObserver> observers = new ArrayList<>();

    public List<Room> search(RoomType type, DateRange range) {
        return rooms.stream()
            .filter(r -> r.type() == type)
            .filter(r -> availability.isAvailable(r, range))
            .toList();
    }

    public Booking book(Customer c, Room r, DateRange range, PaymentStrategy pay) {
        if (!availability.reserve(r, range)) throw new RoomUnavailableException();
        double total = pricing.price(r, range.from(), range.to());
        if (!pay.charge(total)) {
            availability.cancel(r, range);                    // release on failure
            throw new PaymentFailedException();
        }
        Booking b = new Booking(c, r, range, total);
        observers.forEach(o -> o.onConfirmed(b));
        return b;
    }
}
```

## Key design decisions

1. **`RoomAvailabilityManager` is a separate class** — availability querying and locking is a distinct responsibility from `Hotel` or `Room`. SRP win, and an obvious place to add an index/cache.
2. **`Room` is abstract with per-type subclasses** — different room types have different occupancy, amenities, and base prices. Abstract base + subclasses model this cleanly.
3. **Pricing as Strategy** — pricing logic is the most volatile part of the system (seasons, promotions, dynamic). Externalizing it as an injected strategy means a price experiment is a new class, not a change to `Hotel`.
4. **Reserve before charge, release on payment failure** — the *order* matters. If you charge first and the room becomes unavailable while the payment processes, you've taken money for nothing. Reserve first (lock the inventory), then charge, then release on failure.
5. **Date-range overlap check** — the right primitive for availability. A `TreeSet<DateRange>` keeps ranges ordered, so overlap checks can be `O(log n)` with proper comparators.
6. **Observer for notifications** — customer, hotel ops, housekeeping all need to know about bookings. Observer keeps `Hotel.book` from accumulating notification calls inline.

## Extensibility

**Add dynamic pricing (surge based on occupancy):**

1. `class DynamicPricing implements PricingStrategy` — factors in `availability.occupancyRate()`.
2. Inject in place of `SeasonalPricing`. No edits to existing code.

**Add a new room type (PenthouseRoom):**

1. `class PenthouseRoom extends Room` with its own occupancy and amenities.
2. Hotel's search just sees one more `RoomType` value. OCP win.

**Add multi-night discounts:**

1. Wrap the existing pricing strategy in a `DiscountedPricing` decorator that subtracts a percentage for stays ≥ 7 nights.
2. Decorator pattern composes with whatever pricing strategy is current.

**Add a loyalty program:**

1. `Customer` gains a `tier` (Silver/Gold/Platinum).
2. `PricingStrategy.price` takes a context including the customer; strategy applies tier-based discount.
3. Or: implement a `LoyaltyPricing` decorator and chain it.

## Linked concepts

- [OOP](../fundamentals/oop.md) — abstract `Room`, composition of services.
- [SOLID](../fundamentals/solid.md) — SRP for `RoomAvailabilityManager`, OCP via strategies.
- [Design Patterns](../fundamentals/design-patterns.md) — Singleton, Factory, Strategy, Observer.
