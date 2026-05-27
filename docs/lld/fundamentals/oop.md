# Object-Oriented Programming (OOP)

OOP is a way of organizing code around **objects** — bundles of state and behavior that model real-world entities. The four pillars (Encapsulation, Abstraction, Inheritance, Polymorphism) give you the vocabulary; the *judgement* about when to use each is what separates a junior design from a senior one.

## Why it matters

Without OOP discipline, code tends to collapse into giant procedural functions that touch shared mutable state everywhere. A small requirement change ("add a new vehicle type") ripples through dozens of `if/else` blocks. OOP done well gives you:

- **Local reasoning** — a class encapsulates its own invariants, so you don't have to understand the whole system to change a part.
- **Extensibility** — new behavior plugs in via new classes, not by editing old ones (Open/Closed Principle).
- **Testability** — small classes with clear responsibilities can be unit-tested in isolation.

The failure mode that OOP fights against is *"everything depends on everything"* — once you have that, every change is risky.

## Core concepts

### Encapsulation

**Concept**: Bundle data and the methods that operate on that data within a class. Hide internal details behind a controlled interface.

```java
// BAD: public double balance; (anyone can mutate it to -1000000)
// GOOD:
class Account {
    private double balance;

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException();
        this.balance += amount;
    }

    public double getBalance() { return balance; }
}
```

**Why the "good" version is good**: the `Account` class *owns* the invariant "balance never goes negative through deposits". Outside callers cannot bypass the check because they cannot touch the field directly. If tomorrow you need to log every change, you change one method, not 47 call sites.

**Why encapsulation matters**:

- Protects data integrity (invariants are enforced in one place).
- Lets you change internals without breaking clients (you could switch `balance` from `double` to `BigDecimal` without anyone noticing).
- Provides controlled access — read-only fields, validation, lazy computation.

### Abstraction

**Concept**: Hide complex implementation details, expose only the necessary interface.

```java
interface PaymentProcessor {
    boolean processPayment(double amount);
}

class CreditCardProcessor implements PaymentProcessor {
    public boolean processPayment(double amount) {
        // contacts Stripe, handles 3DS, retries on network errors...
        return true;
    }
}

class PayPalProcessor implements PaymentProcessor {
    public boolean processPayment(double amount) {
        // completely different API, different error model
        return true;
    }
}
```

**Why this is good**: the *caller* (a `Checkout` service) only needs to know the contract `processPayment(amount) -> boolean`. It doesn't know about Stripe, PayPal, or any retry logic. Swapping providers becomes a one-line change in the wiring.

**Why abstraction matters**:

- Reduces cognitive load — you reason about *what* something does, not *how*.
- Enables polymorphism (next pillar).
- Makes the code resilient to provider/library changes.

### Inheritance

**Concept**: A child class inherits properties and methods from a parent class, expressing an **IS-A** relationship.

```java
abstract class Vehicle {
    abstract void start();
}

class Car extends Vehicle {
    void start() { /* turn key, fuel pump, ignition */ }
}

class Motorcycle extends Vehicle {
    void start() { /* press button, kick starter */ }
}
```

**When to use inheritance**:

- True **IS-A** relationship — a Car *is* a Vehicle.
- The child genuinely needs all of the parent's behavior.
- The hierarchy is stable (you don't expect to reshuffle it in 6 months).

**When NOT to use inheritance**:

- For pure code reuse — use **composition** instead.
- When the relationship is "has-a" or "uses-a" — that's composition.
- When the child needs to disable parent behavior (LSP violation — see [SOLID](solid.md)).

!!! warning "Prefer composition over inheritance"
    Inheritance is the *tightest* possible coupling in OOP. The child inherits every method, every field, and every quirk of the parent — forever. Composition gives you the same code reuse with a fraction of the coupling.

### Polymorphism

**Concept**: Same interface, different implementations. Treat objects of different concrete types uniformly through their shared abstraction.

```java
List<Shape> shapes = List.of(new Circle(), new Rectangle(), new Triangle());
for (Shape s : shapes) {
    s.draw(); // each class has its own draw() — the right one is called automatically
}
```

**Two flavors**:

1. **Compile-time (Overloading)** — same method name, different parameter lists. `add(int, int)` vs `add(double, double)`. Resolved by the compiler.
2. **Runtime (Overriding)** — child class provides its own version of a parent method. Resolved by the JVM via dynamic dispatch.

**Why polymorphism is the whole point**: it's what enables the Open/Closed Principle. To add `Hexagon`, you write a new class and the loop above just works. No edits to existing code. This is the foundation that makes design patterns like Strategy, State, and Command useful.

## Composition vs Inheritance

- **Inheritance** — IS-A relationship. Tight coupling. Compile-time.
- **Composition** — HAS-A relationship. Loose coupling. Runtime-swappable.

```java
// INHERITANCE: Manager IS-A Employee  (tight coupling)
class Manager extends Employee { /* ... */ }

// COMPOSITION: Manager HAS-A Employee  (loose coupling) — PREFERRED
class Manager {
    private Employee employeeDetails;     // HAS-A
    private List<Employee> team;          // HAS-A many
}
```

**Why composition usually wins**:

- You can change *what* you compose at runtime. A `Car` can swap its `Engine` from `PetrolEngine` to `ElectricEngine`. A `Manager extends Employee` can never stop being an `Employee`.
- You can compose multiple independent behaviors. Java doesn't allow multi-inheritance of classes, but a class can hold references to as many helper objects as needed.
- It naturally encourages small, focused classes (SRP).

!!! tip "Rule of thumb"
    Default to composition. Only reach for inheritance when there is a true IS-A relationship *and* you would be repeating non-trivial code without it.

## Interfaces vs Abstract Classes

| Feature | Interface | Abstract Class |
|---------|-----------|----------------|
| **Methods** | Only abstract (Java 8+ allows default) | Can have concrete methods |
| **Fields** | Only constants (`public static final`) | Can have instance variables |
| **Inheritance** | Class can implement many | Class can extend only one |
| **Constructor** | No constructor | Can have constructor |
| **When to use** | Define a contract (*what to do*) | Share code (*common behavior*) |

**Decision rule**:

- If you only need to declare "objects of type X can do Y" with no shared code → **interface**.
- If multiple subclasses will share real implementation (state, helper methods) → **abstract class**.
- If both — interface for the contract, abstract class for a default implementation. Java's Collections framework does this (`List` interface, `AbstractList` partial implementation, `ArrayList` concrete).

## A worked OOP example: payment processing

Putting the four pillars together on a small domain:

```java
// Abstraction — clients only know the contract.
interface PaymentProcessor {
    PaymentResult process(double amount);
}

// Encapsulation — internal state is private; only safe operations exposed.
abstract class BaseProcessor implements PaymentProcessor {
    private int failureCount = 0;
    private static final int MAX_FAILURES = 3;

    public final PaymentResult process(double amount) {   // Template-method-ish
        if (failureCount >= MAX_FAILURES) return PaymentResult.LOCKED;
        try {
            PaymentResult r = doProcess(amount);
            if (r.isSuccess()) failureCount = 0;
            else failureCount++;
            return r;
        } catch (Exception e) {
            failureCount++;
            throw e;
        }
    }

    protected abstract PaymentResult doProcess(double amount);
}

// Inheritance + Polymorphism — concrete strategies override only the part that varies.
class StripeProcessor extends BaseProcessor {
    protected PaymentResult doProcess(double amount) { /* Stripe SDK */ return PaymentResult.OK; }
}

class PayPalProcessor extends BaseProcessor {
    protected PaymentResult doProcess(double amount) { /* PayPal SDK */ return PaymentResult.OK; }
}

// Polymorphism in action — caller treats all processors uniformly.
class Checkout {
    private final PaymentProcessor processor;
    Checkout(PaymentProcessor p) { this.processor = p; }
    public void pay(double amount) { processor.process(amount); }
}
```

Notice what each pillar contributes:

- **Abstraction**: `Checkout` knows only `PaymentProcessor`. Swapping Stripe for PayPal is a one-line change at the wiring site.
- **Encapsulation**: `failureCount` is private and managed inside `BaseProcessor`. Subclasses cannot accidentally reset it.
- **Inheritance**: `StripeProcessor` and `PayPalProcessor` reuse the failure-tracking logic from `BaseProcessor` without copy-pasting it.
- **Polymorphism**: `process(amount)` resolves to the right `doProcess` at runtime via dynamic dispatch.

This is the *texture* of well-applied OOP: small interfaces, abstract bases for shared infrastructure, concrete classes for variation, dependency injection at the boundary.

## When to use OOP (and when not to)

**Use OOP when**:

- The domain has natural entities with state and behavior (users, orders, vehicles).
- You expect requirements to change — OOP shines at extensibility.
- Multiple teams will collaborate — clear class boundaries make ownership tractable.

**Don't force OOP when**:

- The problem is a pure data transformation pipeline — functional style is often cleaner.
- You're writing a one-off script — classes are overhead, not value.
- The "objects" are anemic (just bags of fields with no behavior) — you've reinvented structs with extra steps.

## Common pitfalls

- **Anemic domain models** — classes with only getters and setters, all logic in "service" classes. You've thrown away encapsulation.
- **Overusing inheritance** — using `extends` for code reuse instead of true IS-A leads to brittle hierarchies and LSP violations.
- **God classes** — one class doing 12 things. Split by responsibility (SRP).
- **Exposed mutable state** — returning a reference to an internal `List` lets callers mutate your state behind your back. Return unmodifiable views or copies.
- **Premature abstraction** — building an `IPaymentProcessor` interface when you have exactly one implementation and no plans for a second. YAGNI.
- **Deep hierarchies** — `class C extends B extends A extends Base`. Hard to reason about, easy to break with LSP violations. Prefer shallow hierarchies plus composition.

## Linked problems

Every LLD problem leans on OOP, but these are particularly good practice for the four pillars:

- [Parking Lot](../problems/parking-lot.md) — abstract `Vehicle` and `ParkingSpot` hierarchies, composition of floors and spots.
- [Chess Game](../problems/chess.md) — `Piece` abstraction with one subclass per piece type, polymorphic `isValidMove`.
- [Vending Machine](../problems/vending-machine.md) — encapsulation of inventory state behind a clean API.
- [File System](../problems/file-system.md) — abstract `Entry` with `File` and `Directory` (Composite pattern).
- [ATM](../problems/atm.md) — abstract `Transaction` with concrete `Withdrawal`/`Deposit`/`Transfer`.
