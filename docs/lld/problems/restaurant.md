# Design a Restaurant Management System

> **Time:** 40–50 minutes

## Problem

Design the software running a dine-in restaurant. The system must:

- Manage **tables** with capacity and status (`FREE`, `RESERVED`, `OCCUPIED`).
- Handle **reservations** for specific dates/times.
- Take **orders** from a table, route them to the kitchen, mark items as ready, deliver to the table.
- Maintain a **menu** with sections and prices, supporting daily specials.
- Generate the **bill** (subtotal, tax, service charge, discounts).
- Distinguish staff roles: **Waiter**, **Chef**, **Manager**.

## Real-world intuition

Restaurants are interesting because the *digital* model has to mirror the *physical* workflow: order is taken → routed to kitchen → cooked → served → billed. Each handoff is an opportunity for state mismatch (kitchen thinks dish is ready, waiter doesn't know yet). A well-designed system makes the workflow explicit and observable.

## Key classes

- `Restaurant` — top-level orchestrator; owns tables, menu, staff, current orders.
- `Table` — number, capacity, status, currently-seated party.
- `Reservation` — links a `Customer`, a `Table`, a time slot, party size.
- `Menu` — sections (Starters, Mains, Desserts) of `MenuItem`s.
- `MenuItem` — name, price, prep time, availability flag.
- `Order` — links a `Table`, list of `OrderItem`s, status (`PLACED`, `IN_PROGRESS`, `READY`, `SERVED`, `BILLED`).
- `OrderItem` — one menu item with quantity and per-item status (`QUEUED`, `COOKING`, `READY`, `SERVED`).
- `Bill` — subtotal, tax, service charge, discount, total.
- `Staff` (abstract) → `Waiter`, `Chef`, `Manager`.

## Design patterns used

- **Observer (Order events)** — kitchen display, waiter terminals, and the manager dashboard all need to see order state changes. Observer keeps `Order.markReady()` clean.
- **Strategy (BillingStrategy)** — service charge varies (some places have it, some don't); tax varies by jurisdiction; discounts come in many forms. Strategy keeps billing rules pluggable.
- **State (Order lifecycle)** — explicit transitions between `PLACED` → `IN_PROGRESS` → `READY` → `SERVED` → `BILLED`, with rules about who can drive each transition. State pattern (or carefully guarded enum transitions) makes invalid states impossible.
- **Factory (OrderFactory)** — creating an order from a list of menu-item selections, validating availability, applying defaults — clean factory candidate.

## Class diagram (sketch)

```text
   ┌──────────────┐
   │ Restaurant   │
   ├──────────────┤
   │ -tables      │◇──▶ Table
   │ -menu        │◇──▶ Menu ──◇──▶ MenuItem
   │ -staff       │◇──▶ Staff (abstract) ─△─ Waiter / Chef / Manager
   │ -orders      │◇──▶ Order ──◇──▶ OrderItem
   └──────────────┘

   Restaurant ──▶ BillingStrategy «interface»
   Order ──▶ OrderObserver «interface» (kitchen display, waiter app)
```

## Code sketch

```java
enum TableStatus { FREE, RESERVED, OCCUPIED }
enum OrderStatus { PLACED, IN_PROGRESS, READY, SERVED, BILLED }
enum ItemStatus  { QUEUED, COOKING, READY, SERVED }

class Table {
    private final int number;
    private final int capacity;
    private TableStatus status = TableStatus.FREE;
}

class MenuItem {
    private final String name;
    private final double price;
    private final Duration prepTime;
    private boolean available = true;
}

class OrderItem {
    private final MenuItem item;
    private final int quantity;
    private ItemStatus status = ItemStatus.QUEUED;
    private final List<OrderObserver> observers;

    public void markCooking() { transition(ItemStatus.COOKING); }
    public void markReady()   { transition(ItemStatus.READY); }
    public void markServed()  { transition(ItemStatus.SERVED); }

    private void transition(ItemStatus next) {
        this.status = next;
        observers.forEach(o -> o.onItemStatusChanged(this));
    }
}

class Order {
    private final Table table;
    private final List<OrderItem> items = new ArrayList<>();
    private OrderStatus status = OrderStatus.PLACED;

    public void addItem(OrderItem oi)   { items.add(oi); }
    public boolean isReady()            { return items.stream().allMatch(i -> i.status() == ItemStatus.READY); }

    public void onItemReady() {
        if (isReady()) {
            this.status = OrderStatus.READY;
        }
    }
}

interface BillingStrategy {
    Bill compute(Order o);
}

class StandardBilling implements BillingStrategy {
    private static final double TAX = 0.05;
    private static final double SERVICE = 0.10;

    public Bill compute(Order o) {
        double subtotal = o.items().stream()
            .mapToDouble(i -> i.item().price() * i.quantity())
            .sum();
        double tax = subtotal * TAX;
        double service = subtotal * SERVICE;
        return new Bill(subtotal, tax, service, 0, subtotal + tax + service);
    }
}

class Restaurant {
    private final List<Table> tables;
    private final Menu menu;
    private BillingStrategy billing = new StandardBilling();

    public Order placeOrder(Table t, List<OrderItem> items) {
        Order o = new Order(t);
        items.forEach(o::addItem);
        // Notify kitchen via the kitchen-display observer registered on each OrderItem.
        return o;
    }

    public Bill bill(Order o) {
        if (o.status() != OrderStatus.SERVED) throw new IllegalStateException();
        Bill b = billing.compute(o);
        o.markBilled();
        return b;
    }
}
```

## Key design decisions

1. **Per-item status, plus overall order status** — diners get their food asynchronously; tracking each item separately lets the kitchen and waiters coordinate without losing partial progress. The order's status is derived from its items' statuses.
2. **Billing as Strategy** — tax rules, service charge norms, and discount campaigns all vary. Inject the billing strategy; standard, no-service-charge, and promotional variants are separate classes.
3. **Observer for order events** — kitchen display subscribes to "item placed", waiter app subscribes to "item ready", manager dashboard subscribes to everything. Loose coupling is essential because the surfaces change frequently.
4. **Tables are mutable, reservations are first-class** — `Table` has a status; `Reservation` is a separate entity that holds a time slot. Don't conflate "this table is reserved at 7pm" with "this table is occupied right now".
5. **Staff hierarchy reflects roles** — waiter, chef, manager have different permitted operations. Modeling them as subclasses (or as roles on a unified `Staff` class) makes authorization explicit.
6. **Order item is its own class, not a tuple** — it carries quantity, status, customizations ("no onions"), and observer hooks. Keeping it small enough to evolve.

## Extensibility

**Add takeout / delivery orders:**

1. `Order` becomes polymorphic: `DineInOrder` (has `Table`), `TakeoutOrder` (has pickup time), `DeliveryOrder` (has address).
2. Billing strategy considers delivery fees.

**Add per-item customizations:**

1. `OrderItem` gains a `Map<String, String> customizations` field.
2. Kitchen display reads them; price strategy might surcharge extras.

**Add a happy-hour discount:**

1. `HappyHourBilling implements BillingStrategy` wraps standard billing with a percentage discount on certain categories during certain hours.
2. Inject conditionally based on current time.

**Add a kitchen capacity model:**

1. Kitchen has a `bandwidth` and a queue; orders flow through it.
2. `OrderItem.markCooking()` blocks until the kitchen has capacity (or queues).

**Add ratings:**

1. After `BILLED`, fire an event to a `RatingObserver` that records customer feedback.
2. Independent of order/billing logic.

## Linked concepts

- [OOP](../fundamentals/oop.md) — staff hierarchy, table/order composition.
- [SOLID](../fundamentals/solid.md) — SRP across `Restaurant`, `Order`, `BillingStrategy`; OCP via strategies.
- [Design Patterns](../fundamentals/design-patterns.md) — Observer, Strategy, State, Factory.
