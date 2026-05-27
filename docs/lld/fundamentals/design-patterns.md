# Design Patterns

Design patterns are **named, reusable solutions** to recurring object-oriented problems. They are not algorithms you copy-paste — they are *templates* of class relationships. Knowing them gives you a shared vocabulary ("let's use a Strategy here") and a head start on common designs.

## Why it matters

Patterns aren't magic. Most of them are obvious in hindsight — they exist because the same shape of problem keeps recurring, and naming the solution makes design discussions tractable. In an interview, the interviewer doesn't want you to *invent* a way to swap payment methods at runtime; they want you to recognize "that's Strategy" and move on.

Patterns are organized into three families:

- **Creational** — how objects are created (Singleton, Factory, Builder, Prototype).
- **Structural** — how objects are composed (Adapter, Decorator, Facade, Proxy).
- **Behavioral** — how objects communicate (Strategy, Observer, Template Method, State, Command).

!!! warning "Patterns are a tool, not a goal"
    Forcing a pattern where it doesn't fit makes code worse, not better. The right question is never *"which pattern should I use?"* — it's *"what's the simplest thing that solves this?"* Patterns are answers that emerge naturally when you ask that question.

## Creational patterns

### Singleton

**Purpose**: Ensure a class has only one instance and provide global access to it.

**When to use**:

- Database connection pools
- Loggers, configuration managers, caches
- Anything that genuinely should be global *and* shared

**Implementation** (thread-safe, lazy):

```java
// Best practice: enum singleton (thread-safe, serialization-safe, no reflection bypass)
public enum DatabaseConnection {
    INSTANCE;

    public Connection getConnection() { /* ... */ return null; }
}

// Usage:
DatabaseConnection.INSTANCE.getConnection();

// Alternative: double-checked locking (older codebases)
public class Logger {
    private static volatile Logger instance;

    private Logger() {}

    public static Logger getInstance() {
        if (instance == null) {
            synchronized (Logger.class) {
                if (instance == null) instance = new Logger();
            }
        }
        return instance;
    }
}
```

!!! warning "Singletons are often overused"
    They introduce hidden global state, make unit testing painful (you can't substitute a mock), and are *not* a substitute for dependency injection. Use singletons only when there is genuinely one shared resource (a hardware device, a process-wide config), and prefer DI for everything else.

### Factory

**Purpose**: Create objects without specifying the exact concrete class. Delegate creation to a factory method.

**When to use**:

- Complex creation logic you don't want repeated.
- The exact type to create is decided at runtime (from user input, config, etc.).
- You want to decouple "what to create" from "how to use it".

**Example**:

```java
interface Vehicle { void drive(); }
class Car implements Vehicle      { public void drive() { /* ... */ } }
class Bike implements Vehicle     { public void drive() { /* ... */ } }
class Truck implements Vehicle    { public void drive() { /* ... */ } }

class VehicleFactory {
    static Vehicle create(String type) {
        return switch (type.toLowerCase()) {
            case "car"   -> new Car();
            case "bike"  -> new Bike();
            case "truck" -> new Truck();
            default      -> throw new IllegalArgumentException("Unknown: " + type);
        };
    }
}

// Caller doesn't need to know which concrete class exists.
Vehicle v = VehicleFactory.create(userChoice);
v.drive();
```

**Why this fits OCP**: the *caller* code never changes when a new vehicle type is added — only the factory grows. Even better: register classes dynamically and the factory itself stops needing edits.

### Builder

**Purpose**: Construct complex objects step by step, separating *construction* from *representation*.

**When to use**:

- Many constructor parameters (especially when most are optional).
- Object creation requires multiple steps or validation.
- You want immutable objects with readable construction code.

**Example**:

```java
public class Pizza {
    private final String size;
    private final boolean cheese, pepperoni, mushrooms, olives;

    private Pizza(Builder b) {
        this.size       = b.size;
        this.cheese     = b.cheese;
        this.pepperoni  = b.pepperoni;
        this.mushrooms  = b.mushrooms;
        this.olives     = b.olives;
    }

    public static class Builder {
        private final String size; // required
        private boolean cheese, pepperoni, mushrooms, olives;

        public Builder(String size)         { this.size = size; }
        public Builder cheese(boolean v)    { this.cheese = v; return this; }
        public Builder pepperoni(boolean v) { this.pepperoni = v; return this; }
        public Builder mushrooms(boolean v) { this.mushrooms = v; return this; }
        public Builder olives(boolean v)    { this.olives = v; return this; }

        public Pizza build() { return new Pizza(this); }
    }
}

// Usage — readable, no telescoping constructors.
Pizza p = new Pizza.Builder("Large")
    .cheese(true)
    .pepperoni(true)
    .mushrooms(true)
    .build();
```

**Why Builder beats "constructor with 8 parameters"**: at the call site, you see *what each value means*. With `new Pizza("Large", true, true, true, false, true)`, you have no idea what those booleans are.

### Prototype

**Purpose**: Create new objects by **copying** (cloning) existing objects rather than constructing from scratch.

**When to use**:

- Object creation is expensive (loaded from disk, computed, fetched over network).
- You want to avoid an explosion of subclasses of a Factory.
- You need a "template" object that gets cloned and tweaked.

```java
public class Document implements Cloneable {
    private String content;
    private List<String> attachments;

    public Document clone() { /* deep copy */ return null; }
}

Document template = loadFromDisk("template.docx");
Document copy = template.clone();   // cheap; no disk hit
copy.setContent("custom content");
```

## Structural patterns

### Adapter

**Purpose**: Convert the interface of an existing class into another interface that clients expect. "I have a thing, the caller wants a slightly different shape, I bridge them."

**When to use**:

- Integrating a legacy library with a modern interface.
- Two third-party libraries with incompatible APIs need to interoperate.

**Real example**: `InputStreamReader` adapts a byte-oriented `InputStream` into a character-oriented `Reader`.

```java
// Adaptee — what we have.
class LegacyXmlReader {
    String readXml() { /* ... */ return null; }
}

// Target — what clients want.
interface DataReader {
    String readJson();
}

// Adapter — translates one into the other.
class XmlToJsonAdapter implements DataReader {
    private final LegacyXmlReader legacy;
    XmlToJsonAdapter(LegacyXmlReader l) { this.legacy = l; }

    public String readJson() {
        String xml = legacy.readXml();
        return convertXmlToJson(xml);
    }

    private String convertXmlToJson(String xml) { /* ... */ return null; }
}
```

### Decorator

**Purpose**: Add new functionality to objects **dynamically** without altering their structure or subclassing.

**When to use**:

- You need to add responsibilities to specific objects, not the whole class.
- Subclassing would cause a combinatorial explosion (`CoffeeWithMilk`, `CoffeeWithMilkAndSugar`, ...).

**Example**:

```java
interface Coffee {
    String description();
    double cost();
}

class SimpleCoffee implements Coffee {
    public String description() { return "Coffee"; }
    public double cost() { return 5.0; }
}

abstract class CoffeeDecorator implements Coffee {
    protected final Coffee wrapped;
    CoffeeDecorator(Coffee c) { this.wrapped = c; }
}

class Milk extends CoffeeDecorator {
    public Milk(Coffee c) { super(c); }
    public String description() { return wrapped.description() + " + milk"; }
    public double cost() { return wrapped.cost() + 1.0; }
}

class Sugar extends CoffeeDecorator {
    public Sugar(Coffee c) { super(c); }
    public String description() { return wrapped.description() + " + sugar"; }
    public double cost() { return wrapped.cost() + 0.5; }
}

// Compose decorations dynamically.
Coffee c = new SimpleCoffee();
c = new Milk(c);
c = new Sugar(c);
System.out.println(c.description() + " = $" + c.cost()); // "Coffee + milk + sugar = $6.5"
```

**Real example**: Java I/O streams. `BufferedInputStream` decorates `FileInputStream` to add buffering; `DataInputStream` adds typed reads on top; etc.

### Facade

**Purpose**: Provide a **simplified interface** to a complex subsystem.

**When to use**:

- A subsystem has many classes and a confusing API.
- You want to define a "happy path" entry point for clients.

```java
// Subsystem — many specialized classes.
class CPU      { void freeze() {}  void execute() {} }
class Memory   { void load(long pos, byte[] data) {} }
class HardDisk { byte[] read(long sector, int size) { return new byte[0]; } }

// Facade — one simple operation: "start the computer".
class ComputerFacade {
    private final CPU cpu = new CPU();
    private final Memory memory = new Memory();
    private final HardDisk disk = new HardDisk();

    public void start() {
        cpu.freeze();
        memory.load(0, disk.read(0, 1024));
        cpu.execute();
    }
}

new ComputerFacade().start();   // client never deals with CPU/Memory/HardDisk directly
```

The facade doesn't *hide* the subsystem — advanced clients can still drop down. It just gives the common case a one-line API.

### Proxy

**Purpose**: Provide a placeholder or surrogate for another object to **control access** to it.

**Types**:

- **Virtual Proxy** — lazy initialization (don't load until needed).
- **Protection Proxy** — access control (check permissions before forwarding).
- **Remote Proxy** — represent an object in a different address space (RPC stubs).

```java
interface Image { void display(); }

class RealImage implements Image {
    private final String filename;
    RealImage(String fn) {
        this.filename = fn;
        loadFromDisk(); // expensive
    }
    public void display() { /* ... */ }
    private void loadFromDisk() { /* ... */ }
}

class ImageProxy implements Image {
    private final String filename;
    private RealImage real;

    ImageProxy(String fn) { this.filename = fn; }

    public void display() {
        if (real == null) real = new RealImage(filename); // lazy load
        real.display();
    }
}
```

## Behavioral patterns

### Strategy

**Purpose**: Define a family of algorithms, encapsulate each one as a class, make them **interchangeable** at runtime.

**When to use**:

- Multiple algorithms solve the same problem differently (sorting, pricing, payment, matching).
- You want to pick the algorithm at runtime.
- You're staring at a long `if/else` or `switch` over a type code.

```java
interface PaymentStrategy {
    void pay(double amount);
}

class CreditCardPayment implements PaymentStrategy {
    public void pay(double amount) { /* charge card */ }
}

class PayPalPayment implements PaymentStrategy {
    public void pay(double amount) { /* PayPal API */ }
}

class UPIPayment implements PaymentStrategy {
    public void pay(double amount) { /* UPI API */ }
}

class ShoppingCart {
    private PaymentStrategy strategy;
    public void setPaymentStrategy(PaymentStrategy s) { this.strategy = s; }
    public void checkout(double total) { strategy.pay(total); }
}

cart.setPaymentStrategy(new CreditCardPayment());
cart.checkout(100.0);
```

**Strategy is the workhorse pattern of LLD interviews.** Almost every problem has at least one "swap this algorithm" axis — pricing, matching, scheduling, validation, fare calculation. Recognize it on sight.

### Observer

**Purpose**: Define a one-to-many dependency. When one object (the **subject**) changes state, all its **observers** are notified automatically.

**When to use**:

- Event-handling systems.
- Model-View separation (MVC).
- Publish/subscribe inside a single process.

```java
interface Observer {
    void update(String news);
}

class NewsAgency {
    private final List<Observer> observers = new ArrayList<>();

    public void subscribe(Observer o)   { observers.add(o); }
    public void unsubscribe(Observer o) { observers.remove(o); }

    public void publish(String news) {
        for (Observer o : observers) o.update(news);
    }
}

class EmailSubscriber implements Observer {
    public void update(String news) { /* send email */ }
}

NewsAgency agency = new NewsAgency();
agency.subscribe(new EmailSubscriber());
agency.publish("Breaking news!"); // every subscriber gets notified
```

### Template Method

**Purpose**: Define the **skeleton** of an algorithm in a base class. Let subclasses override specific steps without changing the overall structure.

**When to use**:

- An algorithm has a fixed sequence of steps, but some steps vary by use case.
- You want to enforce that subclasses follow the same overall flow.

```java
abstract class DataParser {
    // Template method — final, so subclasses cannot reorder the steps.
    public final void parseData() {
        readData();
        processData();
        validateData();
        saveData();
    }

    protected abstract void readData();    // each subclass implements
    protected abstract void processData();

    protected void validateData() { /* default validation, can be overridden */ }
    protected void saveData()     { /* default save logic */ }
}

class CsvParser extends DataParser {
    protected void readData()    { /* read CSV */ }
    protected void processData() { /* parse rows */ }
}

class JsonParser extends DataParser {
    protected void readData()    { /* read JSON */ }
    protected void processData() { /* parse objects */ }
}
```

### State

**Purpose**: Allow an object to **change its behavior** when its internal state changes. The object will appear to change its class.

**When to use**:

- An object's behavior depends heavily on its state (vending machine, ATM, TCP connection).
- You have many conditional statements branching on a `state` field.

**Classic example**: a vending machine with states `NoCoin`, `HasCoin`, `Sold`, `SoldOut`. Each state knows how to react to each event (insert coin, press button, dispense).

```java
interface VendingMachineState {
    void insertCoin(VendingMachine m);
    void pressButton(VendingMachine m);
    void dispense(VendingMachine m);
}

class NoCoinState implements VendingMachineState {
    public void insertCoin(VendingMachine m) { m.setState(new HasCoinState()); }
    public void pressButton(VendingMachine m) { System.out.println("Insert coin first"); }
    public void dispense(VendingMachine m)   { /* nothing */ }
}

class HasCoinState implements VendingMachineState {
    public void insertCoin(VendingMachine m) { System.out.println("Already have coin"); }
    public void pressButton(VendingMachine m) { m.setState(new SoldState()); }
    public void dispense(VendingMachine m)   { /* nothing yet */ }
}

class VendingMachine {
    private VendingMachineState state = new NoCoinState();
    public void setState(VendingMachineState s) { this.state = s; }
    public void insertCoin()  { state.insertCoin(this); }
    public void pressButton() { state.pressButton(this); }
}
```

**Why State beats a giant `switch (currentState)`**: state-specific behavior lives in its own class, transitions are explicit, adding a new state never modifies existing states.

### Command

**Purpose**: Encapsulate a request as an **object**, allowing you to parameterize clients with different requests, queue them, log them, and undo them.

**When to use**:

- Undo/redo functionality.
- Transaction systems.
- Macro commands (compose multiple commands into one).
- Job queues, scheduled tasks.

```java
interface Command {
    void execute();
    void undo();
}

class Light {
    void on()  { /* ... */ }
    void off() { /* ... */ }
}

class LightOnCommand implements Command {
    private final Light light;
    LightOnCommand(Light l) { this.light = l; }

    public void execute() { light.on(); }
    public void undo()    { light.off(); }
}

class RemoteControl {
    private final Deque<Command> history = new ArrayDeque<>();

    public void press(Command c) {
        c.execute();
        history.push(c);
    }

    public void undoLast() {
        if (!history.isEmpty()) history.pop().undo();
    }
}
```

## When to use which pattern

| Problem | Reach for |
|---------|-----------|
| Multiple algorithms for the same task | **Strategy** |
| Behavior depends on internal state with many transitions | **State** |
| Create an object whose type is decided at runtime | **Factory** |
| Constructing an object with many optional fields | **Builder** |
| One object's change must update many others | **Observer** |
| Add behavior to specific instances without subclassing | **Decorator** |
| Need undo/redo, or queued operations | **Command** |
| Tree-like hierarchy where leaves and composites share an interface | **Composite** |
| Wrap a legacy/incompatible API | **Adapter** |
| Simplify access to a complex subsystem | **Facade** |
| Control access, add lazy loading, or proxy a remote call | **Proxy** |
| Fixed algorithm skeleton, varying steps per subclass | **Template Method** |
| Genuinely-global single instance of a resource | **Singleton** |

## Common pitfalls

- **Pattern-matching the interview** — naming patterns without explaining *why* they fit. Interviewers want reasoning, not buzzwords.
- **Singleton-everywhere** — every class becomes a singleton "because it's just one thing". You've reinvented globals.
- **Strategy with one strategy** — if there's only one implementation today and no plausible second, you've added an interface for nothing.
- **State + Strategy confusion** — they look similar (both have polymorphic behaviors). Strategy is *chosen* by the client and stays put; State is *transitioned* internally as the object's state changes.
- **Observer memory leaks** — long-lived subjects holding references to short-lived observers prevent GC. Always provide `unsubscribe`.
- **Builder for 2-parameter objects** — Builder is for *many* parameters. For two, a constructor is fine.
- **Decorators that depend on each other** — if `Sugar` must come after `Milk` for correctness, decoration order should be invisible. Otherwise you've broken the abstraction.

## Linked problems

- [Parking Lot](../problems/parking-lot.md) — Singleton, Strategy, Factory, Observer.
- [Vending Machine](../problems/vending-machine.md) — State (canonical example).
- [Elevator System](../problems/elevator.md) — State + Strategy + Singleton.
- [ATM](../problems/atm.md) — State + Command + Strategy.
- [Chess Game](../problems/chess.md) — Command (Move), State (game status), Strategy (move validation).
- [Notification System](../problems/notification-system.md) — Strategy + Observer + Chain of Responsibility.
- [File System](../problems/file-system.md) — Composite + Visitor.
- [Social Network](../problems/social-network.md) — Observer + Strategy + Composite.
- [Online Shopping](../problems/online-shopping.md) — Strategy + Factory + Observer.
