# Design an Online Shopping System (Amazon-style)

> **Time:** 60+ minutes

## Problem

Design the object model behind an e-commerce platform. The system must:

- Maintain a **product catalog** with categories, prices, descriptions, images, inventory.
- Provide each customer a **shopping cart** they can add/remove from.
- Process **order placement**: validate cart, reserve inventory, charge payment, fulfill.
- Support multiple **payment methods**.
- Manage **inventory** with concurrent-safe decrement.
- Handle **shipping** with multiple carriers and shipping options.
- Track **order status** (`PLACED` → `PAID` → `SHIPPED` → `DELIVERED`, plus `CANCELLED` / `REFUNDED`).

## Real-world intuition

E-commerce is the LLD problem that touches every SOLID principle and most major patterns. Amazon's actual systems are massively distributed — but the *object model* problem at LLD scale is "how do you keep cart, inventory, payment, and shipping decoupled enough that any one of them can change without breaking the others?" The answer is a lot of small services, each owning one concern, plus strategies for the volatile axes.

## Key classes

- `Customer` — profile, addresses, payment methods, order history.
- `Product` — id, name, description, price, category, images, attributes.
- `Inventory` — per-product stock count; safe concurrent decrement.
- `ShoppingCart` — items the customer has selected; quantities; subtotal.
- `CartItem` — product, quantity.
- `Order` — frozen snapshot at checkout: items, prices, shipping, totals, status.
- `OrderItem` — line-item with price-at-order (not live price).
- `PaymentStrategy` (interface) — `CardPayment`, `WalletPayment`, `UpiPayment`, `BNPL`.
- `Shipment` — links an order to a shipping option and tracking number.
- `ShippingStrategy` (interface) — `StandardShipping`, `Express`, `SameDay`, per carrier.
- `OrderService` — orchestrates checkout, payment, fulfillment.
- `Notification` (observer-based) — confirmations, shipping updates, delivery.

## Design patterns used

- **Strategy (PaymentStrategy, ShippingStrategy)** — payments and shipping both have many providers and configurations. Every new gateway or carrier is a new class, no edits to checkout.
- **Factory (PaymentFactory, ShipmentFactory)** — instantiate the right strategy from a string code / user selection. Keeps construction logic in one place.
- **Observer (NotificationService)** — order placed, paid, shipped, delivered — all fan out to email, SMS, push. Decouples notifications from the order lifecycle.
- **State (Order status)** — explicit `PLACED` → `PAID` → `SHIPPED` → `DELIVERED` (plus `CANCELLED`/`REFUNDED`). Each transition has rules about who can drive it and what side effects must occur.
- **Composite (categories of categories)** — product categories are hierarchical (Electronics → Phones → Smartphones). Composite makes the tree uniform.

## Class diagram (sketch)

```text
   ┌──────────┐
   │ Customer │◇──▶ Address, PaymentMethod
   └────┬─────┘
        │ has-a
        ▼
   ┌─────────────┐  *  ┌──────────┐ *  ┌─────────┐
   │ShoppingCart │◇───│ CartItem │◇──│ Product │◇──▶ Category «composite»
   └─────────────┘     └──────────┘    └────┬────┘
                                            │
                                            ▼
                                       ┌──────────┐
                                       │Inventory │
                                       └──────────┘

   Checkout flow:
   ShoppingCart ──▶ OrderService ──▶ Order ──◇── OrderItem (frozen price)
                          │
                          ├─▶ PaymentStrategy «interface»
                          ├─▶ ShippingStrategy «interface»
                          └─▶ NotificationService «observer-hub»

   Order: PLACED → PAID → SHIPPED → DELIVERED  (or CANCELLED / REFUNDED)
```

## Code sketch

```java
class Product {
    private final String id;
    private final String name;
    private double price;
    private Category category;
}

class Inventory {
    private final Map<String, Integer> stock = new ConcurrentHashMap<>();

    public synchronized boolean reserve(String productId, int qty) {
        int avail = stock.getOrDefault(productId, 0);
        if (avail < qty) return false;
        stock.put(productId, avail - qty);
        return true;
    }
    public synchronized void release(String productId, int qty) {
        stock.merge(productId, qty, Integer::sum);
    }
}

class ShoppingCart {
    private final Customer customer;
    private final Map<String, CartItem> items = new LinkedHashMap<>();

    public void add(Product p, int qty) {
        items.merge(p.id(), new CartItem(p, qty), (a, b) -> new CartItem(p, a.qty() + b.qty()));
    }
    public double subtotal() {
        return items.values().stream().mapToDouble(i -> i.product().price() * i.qty()).sum();
    }
}

interface PaymentStrategy {
    boolean charge(Customer c, double amount);
}

class CardPayment implements PaymentStrategy {
    public boolean charge(Customer c, double amount) { /* gateway */ return true; }
}

interface ShippingStrategy {
    Shipment createShipment(Order o, Address dest);
    double cost(Order o, Address dest);
}

class StandardShipping implements ShippingStrategy {
    public Shipment createShipment(Order o, Address dest) { /* ... */ return null; }
    public double cost(Order o, Address dest)             { return 5.0; }
}

enum OrderStatus { PLACED, PAID, SHIPPED, DELIVERED, CANCELLED, REFUNDED }

class Order {
    private final Customer customer;
    private final List<OrderItem> items;            // frozen snapshot
    private final double total;
    private OrderStatus status = OrderStatus.PLACED;
    private Shipment shipment;

    public void markPaid()      { transition(OrderStatus.PAID); }
    public void markShipped(Shipment s) { this.shipment = s; transition(OrderStatus.SHIPPED); }
    public void markDelivered() { transition(OrderStatus.DELIVERED); }
    public void cancel() {
        if (status == OrderStatus.DELIVERED) throw new IllegalStateException();
        transition(OrderStatus.CANCELLED);
    }

    private void transition(OrderStatus next) {
        // validate transition is legal
        this.status = next;
    }
}

class OrderService {
    private final Inventory inventory;
    private final List<OrderObserver> observers;

    public Order placeOrder(ShoppingCart cart, PaymentStrategy pay, ShippingStrategy ship, Address dest) {
        // 1. Reserve inventory atomically.
        List<String> reserved = new ArrayList<>();
        try {
            for (CartItem it : cart.items()) {
                if (!inventory.reserve(it.product().id(), it.qty())) {
                    throw new OutOfStockException(it.product().id());
                }
                reserved.add(it.product().id());
            }
            // 2. Snapshot prices into an Order.
            Order order = freezeOrder(cart, ship, dest);
            // 3. Charge.
            if (!pay.charge(cart.customer(), order.total())) {
                throw new PaymentFailedException();
            }
            order.markPaid();
            // 4. Create shipment.
            Shipment s = ship.createShipment(order, dest);
            order.markShipped(s);
            observers.forEach(o -> o.onPlaced(order));
            return order;
        } catch (Exception e) {
            // Compensating release on failure.
            reserved.forEach(id -> inventory.release(id, qtyFor(cart, id)));
            throw e;
        }
    }
    private Order freezeOrder(ShoppingCart cart, ShippingStrategy ship, Address dest) { /* ... */ return null; }
    private int qtyFor(ShoppingCart cart, String id) { /* ... */ return 0; }
}
```

## Key design decisions

1. **Snapshot prices at order time, not order display** — `OrderItem` stores `priceAtOrder`. Otherwise a later price change retroactively edits old orders.
2. **Reserve before charge, release on failure** — same pattern as hotel booking. Lock the inventory before risking a payment; if the payment fails, release. Out of order, you risk taking money for stock that's gone.
3. **Inventory is its own class with synchronized decrement** — concurrent checkouts on the last unit of a popular product is the classic race condition. Centralized atomic reserve/release is the right primitive.
4. **Strategy for payment AND shipping** — both axes change frequently and independently. Different orders use different combinations.
5. **State on `Order` is enum with guarded transitions** — `cancel()` from `DELIVERED` should fail loud. Encoding rules in `transition()` makes invalid states impossible.
6. **Observer for everything async** — confirmation email, inventory restock alert, fraud check, recommendation re-training. None of these belong in `OrderService.placeOrder`.
7. **Category as Composite** — categories nest; product belongs to a leaf category but is discoverable from any ancestor. Composite avoids special-casing the tree walk.

## Extensibility

**Add a new payment method (e.g., BNPL):**

1. `class BnplPayment implements PaymentStrategy` calling the BNPL provider.
2. Wire into the UI selector. Order flow unchanged.

**Add a new carrier (e.g., local courier):**

1. `class LocalCourierShipping implements ShippingStrategy`.
2. Available zip codes determine when it's offered.

**Add coupons / promotions:**

1. `Coupon` entity; `CouponStrategy` to compute discount on a cart.
2. `OrderService.placeOrder` consults coupon strategy before payment.

**Add returns / refunds:**

1. `Order` gains `markReturned()` and `markRefunded()` transitions.
2. Refund processing dispatched through the same `PaymentStrategy` (most providers have a `refund` API).

**Add recommendations / "frequently bought together":**

1. `PurchaseObserver` builds co-occurrence statistics on each `onPlaced`.
2. Product detail pages consult the recommendation service.

**Add inventory across multiple warehouses:**

1. `Inventory` becomes `MultiWarehouseInventory` with per-warehouse stock.
2. `reserve` picks the best warehouse (closest, cheapest).
3. Public interface unchanged; `OrderService` doesn't know about warehouses.

## Linked concepts

- [OOP](../fundamentals/oop.md) — composition of cart/order/shipment.
- [SOLID](../fundamentals/solid.md) — touches all five principles: SRP (one class per concern), OCP (new payments, shipping), LSP (strategies are substitutable), ISP (small strategy interfaces), DIP (inject strategies into orchestrator).
- [Design Patterns](../fundamentals/design-patterns.md) — Strategy, Factory, Observer, State, Composite.
