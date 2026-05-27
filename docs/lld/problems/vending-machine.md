# Design a Vending Machine

> **Time:** 30–40 minutes

## Problem

Design the software for a vending machine that sells canned drinks and snacks. The system must:

- **Display items** and their prices.
- **Accept coins/notes** (with multiple denominations).
- **Dispense items** when enough money has been inserted and a valid item is selected.
- **Return change** if the inserted amount exceeds the price.
- Track **inventory** and refuse selections for sold-out items.
- Handle the **return coin** button — cancel current transaction, return all inserted money.

## Real-world intuition

The vending machine is *the* canonical State pattern example. Every input (insert coin, select item, press return) means something different depending on the current phase: no coins yet, partial money inserted, item dispensing, sold out. A naive implementation devolves into a tangle of conditionals; the State pattern keeps each phase's behavior in its own class.

## Key classes

- `VendingMachine` — the machine; holds the current state, inventory, balance inserted so far.
- `VendingMachineState` (interface) — `IdleState` (no coins), `HasCoinsState`, `DispensingState`, `SoldOutState`.
- `Item` — name, price, slot id; mostly immutable.
- `Inventory` — map of slot → (item, count). Encapsulates stock changes.
- `CoinDispenser` — physical change machine; knows available denominations and how to make change.

## Design patterns used

- **State (VendingMachineState)** — the core pattern. Every input (`insertCoin`, `selectItem`, `pressReturn`) is handled differently in each state. State objects keep each phase isolated and make transitions explicit. Adding a new state (e.g., `MaintenanceState`) is one new class.
- **Singleton (VendingMachine)** — one machine, one process. Genuine global hardware resource.
- **Strategy (optional: ChangeStrategy)** — if you want pluggable algorithms for making change (greedy, balanced, prefer-coins), Strategy fits.

## Class diagram (sketch)

```text
       ┌───────────────────┐
       │  VendingMachine   │ «Singleton»
       ├───────────────────┤
       │ -state            │──▶ VendingMachineState «interface»
       │ -inventory        │            △
       │ -insertedAmount   │            ┊
       │ -coinDispenser    │      Idle  HasCoins  Dispensing  SoldOut
       └───────────────────┘

   VendingMachine ──◆── Inventory ──◆── (slot → Item, count)
   VendingMachine ──◆── CoinDispenser
```

## Code sketch

```java
interface VendingMachineState {
    void insertCoin(VendingMachine vm, Coin c);
    void selectItem(VendingMachine vm, String slot);
    void pressReturn(VendingMachine vm);
}

class IdleState implements VendingMachineState {
    public void insertCoin(VendingMachine vm, Coin c) {
        vm.addCoin(c);
        vm.setState(new HasCoinsState());
    }
    public void selectItem(VendingMachine vm, String slot) { vm.display("Insert coins first"); }
    public void pressReturn(VendingMachine vm)             { /* nothing to return */ }
}

class HasCoinsState implements VendingMachineState {
    public void insertCoin(VendingMachine vm, Coin c) {
        vm.addCoin(c);     // accumulate
    }
    public void selectItem(VendingMachine vm, String slot) {
        Item item = vm.getInventory().peek(slot);
        if (item == null) { vm.display("Sold out"); return; }
        if (vm.getInserted() < item.price()) {
            vm.display("Need " + (item.price() - vm.getInserted()) + " more");
            return;
        }
        vm.setState(new DispensingState());
        vm.dispense(slot, item);
    }
    public void pressReturn(VendingMachine vm) {
        vm.returnCoins();
        vm.setState(new IdleState());
    }
}

class DispensingState implements VendingMachineState {
    public void insertCoin(VendingMachine vm, Coin c)      { vm.display("Wait, dispensing"); }
    public void selectItem(VendingMachine vm, String slot) { vm.display("Wait, dispensing"); }
    public void pressReturn(VendingMachine vm)             { vm.display("Wait, dispensing"); }
}

class VendingMachine {
    private static final VendingMachine INSTANCE = new VendingMachine();
    private VendingMachineState state = new IdleState();
    private final Inventory inventory = new Inventory();
    private final CoinDispenser coinDispenser = new CoinDispenser();
    private double insertedAmount = 0;

    public static VendingMachine get() { return INSTANCE; }

    public void insertCoin(Coin c)   { state.insertCoin(this, c); }
    public void selectItem(String s) { state.selectItem(this, s); }
    public void pressReturn()        { state.pressReturn(this); }

    void addCoin(Coin c)             { insertedAmount += c.value(); }
    void setState(VendingMachineState s) { this.state = s; }
    double getInserted()             { return insertedAmount; }
    Inventory getInventory()         { return inventory; }

    void dispense(String slot, Item item) {
        inventory.take(slot);
        double change = insertedAmount - item.price();
        coinDispenser.giveChange(change);
        insertedAmount = 0;
        setState(inventory.isEmpty() ? new SoldOutState() : new IdleState());
    }

    void returnCoins() {
        coinDispenser.giveChange(insertedAmount);
        insertedAmount = 0;
    }
}
```

## Key design decisions

1. **State pattern is non-negotiable here** — this problem exists in textbooks specifically to teach the State pattern. Solving it without State is the wrong answer.
2. **Inserted amount lives on the machine, not the state** — state is *behavior*, not data. The accumulating balance and inventory belong to the machine; states *act on* them.
3. **`Inventory` is its own class** — knowing stock counts, peeking without taking, atomic decrement on dispense. SRP win.
4. **`CoinDispenser` separate from inventory** — change-making is a different responsibility (different denominations, different state) than item stock.
5. **State transitions are explicit** — every state's methods either no-op, error, or call `setState`. There is no implicit "the state changes because some field changed" — auditability matters.
6. **Sold-out is a state, not a check** — once everything is gone, all inputs are rejected uniformly. A `SoldOutState` makes that pattern obvious; flag-based checks scatter the same logic across methods.

## State diagram

```text
                  insertCoin
       ┌──────────────────────────────┐
       ▼                              │
   ┌──────┐  insertCoin   ┌──────────┐
   │ Idle │──────────────▶│ HasCoins │
   └──────┘               └────┬─────┘
       ▲                       │ selectItem(price reached)
       │                       ▼
       │                 ┌────────────┐
       │                 │ Dispensing │
       │                 └─────┬──────┘
       │                       │ done
       │  inventory not empty  │
       └──────────────────┬────┘
                          │ inventory empty
                          ▼
                     ┌─────────┐
                     │ SoldOut │  (rejects all inputs until refilled)
                     └─────────┘

   HasCoins ──pressReturn──▶ Idle (refunds inserted)
```

## Extensibility

**Add a new state (Maintenance):**

1. `class MaintenanceState implements VendingMachineState` — refuses customer inputs, accepts operator commands.
2. Transition triggered by maintenance key. Existing states untouched.

**Add cashless payment (card / UPI):**

1. Introduce `PaymentSource` interface; `Coin` and `CardCharge` both implement it.
2. The machine accumulates "credit" from any source. State logic doesn't change.

**Add a return-bottle-for-refund flow:**

1. New state `BottleReturnState`, new event `insertBottle`. Existing states gain a no-op `insertBottle`.
2. State pattern absorbs the new event cleanly.

**Add per-slot pricing rules (happy hour):**

1. `Item.price()` becomes `Item.priceAt(Instant)` or accepts a `PricingStrategy`.
2. State logic still just compares inserted vs price.

## Linked concepts

- [OOP](../fundamentals/oop.md) — `Item`, `Inventory`, machine state encapsulation.
- [SOLID](../fundamentals/solid.md) — SRP across machine/inventory/dispenser, OCP via new states.
- [Design Patterns](../fundamentals/design-patterns.md) — State (canonical), Singleton, optional Strategy for change-making.
