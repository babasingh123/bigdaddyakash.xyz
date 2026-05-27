# SOLID Principles

SOLID is five design principles that, applied consistently, push object-oriented code toward being **maintainable, extensible, and testable**. They are guidelines, not laws — but in an interview, every refactor you suggest should trace back to one of these.

## Why it matters

The cost of code is not writing it once — it's *changing* it under pressure six months later. SOLID is a checklist for "is this design going to hurt the next person who touches it?" Code that ignores SOLID typically exhibits:

- **Shotgun surgery** — one tiny change forces edits in a dozen files.
- **Fragility** — fixing a bug breaks something seemingly unrelated.
- **Rigidity** — adding a feature requires modifying classes that "should" be stable.
- **Untestability** — you can't instantiate a class without instantiating its database, network, and clock.

Each SOLID principle attacks one of these failure modes.

## Core concepts

### S — Single Responsibility Principle (SRP)

**Definition**: A class should have only one reason to change. Each class should do one thing well.

```java
// BAD: Employee class handles data, persistence, AND reporting.
// Three orthogonal reasons to change: business rules, DB schema, report format.
class Employee {
    void calculatePay()      { /* business rules */ }
    void saveToDatabase()    { /* JDBC */ }
    void generateReport()    { /* PDF generation */ }
}

// GOOD: separate responsibilities, each class has one reason to change.
class Employee                { /* data only */ }
class PayrollCalculator       { double calculatePay(Employee e) { /* ... */ return 0; } }
class EmployeeRepository      { void save(Employee e) { /* ... */ } }
class ReportGenerator         { void generateReport(Employee e) { /* ... */ } }
```

**Why this matters**:

- The HR team's pay-rule changes don't risk breaking the report layout.
- The DBA's schema migration touches only `EmployeeRepository`.
- You can unit-test `PayrollCalculator` without spinning up a database.

!!! tip "Sniff test for SRP"
    Ask: *"if I rename this class to describe everything it does, does the name contain 'and'?"* `EmployeeAndPayrollAndReporting` is a smell.

### O — Open/Closed Principle (OCP)

**Definition**: Classes should be **open for extension** but **closed for modification**.

```java
// BAD: must modify the class to add any new discount type.
class DiscountCalculator {
    double calc(String type, double amount) {
        if (type.equals("PERCENTAGE")) return amount * 0.9;
        else if (type.equals("FIXED")) return amount - 10;
        // Adding "SEASONAL" requires editing this method, risking the others.
    }
}

// GOOD: open for extension, closed for modification.
interface DiscountStrategy {
    double calculate(double amount);
}

class PercentageDiscount implements DiscountStrategy {
    public double calculate(double amount) { return amount * 0.9; }
}

class FixedDiscount implements DiscountStrategy {
    public double calculate(double amount) { return amount - 10; }
}

// Adding a new discount type = new class. Existing code never changes.
class SeasonalDiscount implements DiscountStrategy {
    public double calculate(double amount) { return amount * 0.75; }
}
```

**Why this matters**:

- Existing, working, tested code stays untouched — no regression risk.
- New behavior arrives as new files in pull requests, not as scattered `else if` additions.
- This is exactly what enables the Strategy, State, and Factory patterns.

!!! warning "OCP is the principle most often violated by switch/if-chains"
    Whenever you see a `switch` on a string or enum determining behavior, ask: *"would this be cleaner as polymorphism?"* The answer is usually yes once there are 3+ cases.

### L — Liskov Substitution Principle (LSP)

**Definition**: Objects of a superclass should be replaceable with objects of any subclass without breaking the application.

```java
// BAD: Ostrich violates the contract that Birds can fly.
class Bird {
    void fly() { /* default flying behavior */ }
}

class Ostrich extends Bird {
    void fly() { throw new UnsupportedOperationException("Can't fly!"); }
    // Now any code that calls bird.fly() can blow up at runtime if it's an Ostrich.
}

// GOOD: model the hierarchy honestly.
abstract class Bird {
    abstract void move();
}

class FlyingBird extends Bird {
    void move() { fly(); }
    void fly() { /* ... */ }
}

class Ostrich extends Bird {
    void move() { run(); }
    void run() { /* ... */ }
}
// bird.move() works for every bird, no surprises.
```

**Why this matters**:

- LSP violations are a sign that your inheritance hierarchy doesn't match reality.
- They make polymorphism unsafe — callers can't actually treat all subclasses uniformly.
- The classic Square-extends-Rectangle problem is the same trap: setters that work on Rectangle break the invariants of Square.

!!! tip "LSP sniff test"
    If you find yourself writing `if (x instanceof Subclass)` to handle special cases, the subclass probably violates LSP. Either it doesn't belong in the hierarchy, or the hierarchy needs to be reshaped.

### I — Interface Segregation Principle (ISP)

**Definition**: No client should be forced to depend on methods it doesn't use. Prefer many small, role-specific interfaces over one large general-purpose interface.

```java
// BAD: fat interface forces irrelevant methods on implementers.
interface Worker {
    void work();
    void eat();
    void sleep();
}

class RobotWorker implements Worker {
    public void work() { /* ... */ }
    public void eat() { throw new UnsupportedOperationException(); }   // smell
    public void sleep() { throw new UnsupportedOperationException(); } // smell
}

// GOOD: segregated interfaces, each implementer picks what's relevant.
interface Workable { void work(); }
interface Eatable  { void eat(); }
interface Sleepable { void sleep(); }

class HumanWorker implements Workable, Eatable, Sleepable {
    public void work() { /* ... */ }
    public void eat() { /* ... */ }
    public void sleep() { /* ... */ }
}

class RobotWorker implements Workable {
    public void work() { /* ... */ }
}
```

**Why this matters**:

- Implementers aren't forced into nonsense methods (`throw UnsupportedOperation` is a code smell).
- Callers depend only on the slice of behavior they need — easier to mock, easier to evolve.
- Adding a new capability (`Chargeable` for robots) doesn't force every existing worker to implement it.

### D — Dependency Inversion Principle (DIP)

**Definition**:

1. **High-level modules should not depend on low-level modules.** Both should depend on **abstractions**.
2. **Abstractions should not depend on details.** Details should depend on abstractions.

```java
// BAD: high-level UserService directly instantiates and depends on a concrete DB.
class UserService {
    private MySQLDatabase db = new MySQLDatabase();
    // Tightly coupled. Can't test without MySQL. Can't switch to Postgres without surgery.
}

// GOOD: both UserService and MySQLDatabase depend on the Database abstraction.
interface Database {
    void save(String data);
}

class MySQLDatabase implements Database {
    public void save(String data) { /* JDBC */ }
}

class PostgreSQLDatabase implements Database {
    public void save(String data) { /* different JDBC */ }
}

class UserService {
    private final Database db;
    UserService(Database db) { this.db = db; } // injected — caller decides which DB
}
```

**Why this matters**:

- Swap implementations at runtime — real DB in production, in-memory fake in tests.
- High-level business logic stops being coupled to low-level infrastructure.
- DI frameworks (Spring, Guice) exist *because* DIP is so universally useful — they automate the wiring.

!!! tip "DIP in interviews"
    Whenever a class instantiates another class with `new` inside its body, ask: *"could I inject this instead?"* Constructor injection is the simplest, most testable form.

## When to use SOLID

- **Always when designing for change** — anything that will live longer than a sprint.
- **In code reviews** — SOLID gives you precise language for "this design is going to hurt later".
- **In LLD interviews** — every refactor or design choice should cite a SOLID principle when challenged.

## When SOLID is overkill

- **Throwaway scripts** — a 50-line data-massaging script doesn't need DI and 4 interfaces.
- **Stable, simple domains** — if there is genuinely only one implementation and no plausible second one, an interface is just noise (YAGNI beats DIP in those cases).
- **Performance-critical hot paths** — virtual dispatch and indirection can matter; sometimes a concrete class is correct.

## Common pitfalls

- **Splitting SRP too finely** — one class per method is not SRP, it's a mess. The "one reason to change" must be at the right level of granularity.
- **OCP cargo-cult** — adding a strategy interface for a behavior that has exactly one implementation and no future. That's not OCP, that's wasted complexity.
- **LSP violations through "is-a-kind-of" instead of "is-a"** — Square is-a-kind-of Rectangle in geometry class, but Square *substituting for* Rectangle in code breaks. Be ruthless about what "IS-A" really means in your codebase.
- **ISP applied to data interfaces** — splitting `User` into `Namable`, `Ageable`, etc. is silly. ISP is about *behavior* (methods you call), not data shape.
- **DIP without DI** — declaring an interface and then writing `new ConcreteThing()` inside the consumer defeats the purpose. The dependency must come *from outside* (constructor, setter, framework).

## Linked problems

- [Parking Lot](../problems/parking-lot.md) — SRP (split `ParkingLot`, `Floor`, `Spot`), OCP (new vehicle/spot types), DIP (Payment strategy).
- [Online Shopping](../problems/online-shopping.md) — touches all five principles.
- [Notification System](../problems/notification-system.md) — OCP via Strategy + Chain of Responsibility.
- [Elevator System](../problems/elevator.md) — SRP and State transitions.
- [ATM](../problems/atm.md) — DIP (inject `Bank`, `CashDispenser`), Command for transactions.
