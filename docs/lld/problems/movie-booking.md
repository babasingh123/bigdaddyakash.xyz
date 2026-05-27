# Design a Movie Ticket Booking System

> **Time:** 40–50 minutes

## Problem

Design the software behind a movie ticket booking app (BookMyShow, Fandango). The system must:

- Manage multiple **cinemas**, each with multiple **screens**.
- Schedule **shows** of **movies** at specific times on specific screens.
- Allow customers to **select seats** for a show, with real-time availability.
- Support multiple **payment** methods.
- Handle **cancellation** with refund rules.
- Apply **pricing** that varies by show time (matinee/evening), day of week, and seat category.

## Real-world intuition

The defining challenge is **concurrent seat selection**: two users browsing the same show at the same time must not both succeed at booking seat F12. The right primitive is short-lived reservations ("hold" the seat for 5 minutes while the user enters card details) backed by atomic compare-and-set on the seat's status. This pattern reappears in airline seating, concert tickets, restaurant reservations — it's the same problem.

## Key classes

- `Cinema` — physical location; owns screens.
- `Screen` — one auditorium; owns a fixed seat layout.
- `Seat` — single seat: row, column, category (`STANDARD`, `PREMIUM`, `RECLINER`).
- `Movie` — title, duration, language, certification.
- `Show` — a (movie, screen, start time) triple; owns the per-seat status for that show.
- `Booking` — links a `Customer`, a `Show`, a set of `Seat`s; status (`PENDING_PAYMENT`, `CONFIRMED`, `CANCELLED`).
- `PricingStrategy` — computes ticket price for (show, seat).
- `PaymentStrategy` — payment provider abstraction.
- `BookingService` — orchestrates seat-hold, payment, and confirmation.

## Design patterns used

- **Strategy (PricingStrategy)** — pricing varies on multiple independent axes (time of day, day of week, seat category, demand). Strategy is the right abstraction; concrete classes like `MatineePricing`, `WeekendPricing`, `DemandPricing` can stack via decoration.
- **Observer (BookingObserver)** — notifications to the customer and analytics on each booking event. Decouples notification from booking logic.
- **Factory (BookingFactory or BookingBuilder)** — assembling a `Booking` requires combining a show, seats, customer, total amount — clean fit for a factory or builder.
- **State (Booking.status)** — `PENDING_PAYMENT` → `CONFIRMED` / `EXPIRED`; `CONFIRMED` → `CANCELLED`. State transitions deserve explicit handling.

## Class diagram (sketch)

```text
   ┌────────┐ 1   * ┌────────┐ 1   * ┌────────┐
   │ Cinema │◆─────│ Screen │◆─────│  Seat  │
   └────────┘       └───┬────┘       └────────┘
                        │ 1
                        │ *
                    ┌───▼────┐         ┌────────┐
                    │  Show  │◇───────│ Movie  │
                    └───┬────┘         └────────┘
                        │ shown seats (status per show)
                        │
                    ┌───▼─────┐
                    │ Booking │ «state-driven»
                    ├─────────┤
                    │ customer│
                    │ seats   │
                    │ payment │
                    └─────────┘

   BookingService ──▶ PricingStrategy «interface»
   BookingService ──▶ PaymentStrategy «interface»
   BookingService ──▶ BookingObserver «interface»
```

## Code sketch

```java
enum SeatStatus { AVAILABLE, HELD, BOOKED }
enum SeatCategory { STANDARD, PREMIUM, RECLINER }

class Show {
    private final Movie movie;
    private final Screen screen;
    private final LocalDateTime startTime;

    // Per-show seat status. Concurrent map for thread-safety.
    private final Map<String, SeatHold> seatStatus = new ConcurrentHashMap<>();

    record SeatHold(SeatStatus status, String holdToken, Instant expiresAt) {}

    public synchronized boolean tryHold(List<Seat> seats, String token, Duration ttl) {
        // Atomic check-and-set across the requested seats.
        for (Seat s : seats) {
            SeatHold cur = seatStatus.get(s.id());
            if (cur != null && cur.status() != SeatStatus.AVAILABLE) return false;
        }
        Instant exp = Instant.now().plus(ttl);
        for (Seat s : seats) {
            seatStatus.put(s.id(), new SeatHold(SeatStatus.HELD, token, exp));
        }
        return true;
    }

    public void confirm(List<Seat> seats, String token) {
        for (Seat s : seats) {
            SeatHold cur = seatStatus.get(s.id());
            if (cur == null || !token.equals(cur.holdToken())) {
                throw new IllegalStateException("Hold expired or hijacked");
            }
            seatStatus.put(s.id(), new SeatHold(SeatStatus.BOOKED, token, null));
        }
    }
}

interface PricingStrategy {
    double price(Show show, Seat seat);
}

class TieredPricing implements PricingStrategy {
    public double price(Show show, Seat seat) {
        double base = switch (seat.category()) {
            case STANDARD -> 200;
            case PREMIUM  -> 400;
            case RECLINER -> 600;
        };
        return base * timeMultiplier(show.startTime());
    }
    private double timeMultiplier(LocalDateTime t) { /* matinee 0.8x, evening 1.2x */ return 1.0; }
}

class BookingService {
    private final PricingStrategy pricing;
    private final List<BookingObserver> observers;

    public Booking book(Customer c, Show show, List<Seat> seats, PaymentStrategy pay) {
        String token = UUID.randomUUID().toString();
        if (!show.tryHold(seats, token, Duration.ofMinutes(5))) {
            throw new SeatsUnavailableException();
        }
        double total = seats.stream().mapToDouble(s -> pricing.price(show, s)).sum();
        if (!pay.charge(c, total)) {
            show.releaseHold(seats, token);
            throw new PaymentFailedException();
        }
        show.confirm(seats, token);
        Booking b = new Booking(c, show, seats, total);
        observers.forEach(o -> o.onBooked(b));
        return b;
    }
}
```

## Key design decisions

1. **Per-show seat status, not per-screen** — the same physical seat is `AVAILABLE` for one show and `BOOKED` for another. Status lives on the `Show`, not the `Seat`. The `Seat` is just a coordinate + category.
2. **Two-phase seat reservation (hold → confirm)** — a `HELD` state with a TTL prevents the classic race "I selected seats; someone else bought them while I typed my card". The hold token defends against another browser session confirming a hold it doesn't own.
3. **Concurrent map + per-show lock** — hold logic must be atomic across multiple seats. Synchronizing on the `Show` is fine-grained enough (different shows don't block each other) and keeps the code simple.
4. **Strategy for pricing, composable** — matinee discount, weekend surge, and demand-based pricing can all be independent strategies stacked via decoration. Avoids a giant pricing switch.
5. **Observer for notifications** — emails, push, and analytics on each booking event. Inline calls would clutter `BookingService`.
6. **`Booking` has explicit state** — `PENDING_PAYMENT`, `CONFIRMED`, `EXPIRED`, `CANCELLED`. Transitions are constrained (you can't un-cancel) and easy to test.

## Extensibility

**Add a new pricing rule (e.g., student discount):**

1. Implement `class StudentPricing implements PricingStrategy`, or decorate the existing strategy.
2. Inject. No edits to booking flow.

**Add a new payment method (e.g., Wallet):**

1. `class WalletPayment implements PaymentStrategy`.
2. Wire into the UI selector. Booking service unchanged.

**Add seat selection by group preference (e.g., 4 seats together):**

1. Add `Screen.findContiguous(int n, SeatCategory)` that searches the layout.
2. Booking flow consumes the proposed group of seats and uses the same hold-confirm flow.

**Add a recommendation engine ("you might like these shows"):**

1. Subscribe a `RecommendationObserver` to booking events.
2. It updates its model on each `onBooked(b)`. Booking logic unaware.

## Linked concepts

- [OOP](../fundamentals/oop.md) — composition of cinema/screen/show.
- [SOLID](../fundamentals/solid.md) — OCP via pricing strategies, SRP across `Show`, `BookingService`, `PaymentStrategy`.
- [Design Patterns](../fundamentals/design-patterns.md) — Strategy, Observer, Factory, State.
