# Design a Library Management System

> **Time:** 30–40 minutes

## Problem

Design the software for a library that lends physical books. The system must:

- Allow librarians to **add / remove books** from the catalog.
- Allow members to **issue (borrow)** books and **return** them.
- Track **due dates** and compute **fines** on overdue returns.
- Support **search** by title, author, and ISBN.
- Handle **multiple physical copies** of the same logical book.
- Optionally support **book reservation** (hold the next available copy).

## Real-world intuition

University libraries, public libraries, and corporate book exchanges all face the same modeling challenge: a "book" is two things at once — the *abstract title* (information about the work) and the *physical copy* (a specific item that can be checked out). Conflating them is the most common modeling mistake on this problem; separating them is what makes the design clean.

## Key classes

- `Library` — singleton; entry point; owns catalog and members.
- `Book` — the *logical* book: title, author, ISBN, subject. There is one `Book` per ISBN.
- `BookItem` — a *physical copy* of a `Book`. Has a barcode and a status (`AVAILABLE`, `LOANED`, `RESERVED`, `LOST`). Issued and returned individually.
- `Member` — library patron; can borrow up to N books at once.
- `Librarian` — staff role; can add/remove books, override holds, manage fines.
- `Catalog` — search index over `Book`s by title/author/ISBN; abstracted to allow swapping in a real search engine later.
- `Loan` — record of one borrow event: which `BookItem`, which `Member`, due date, return date.
- `Fine` — accrued amount on a `Loan` when overdue; computed by a `FineStrategy`.
- `Reservation` — request to hold the next available copy of a `Book`.

## Design patterns used

- **Singleton (Library)** — there is one library per deployment; staff and members alike act through it. Acceptable because it really is process-wide global state.
- **Factory (MemberFactory)** — creating a new `Member` involves validating an ID type, assigning a barcode, generating credentials. A factory hides those steps behind a clean call.
- **Strategy (FineStrategy)** — overdue fine rules vary (per-day flat, escalating, capped). Strategy lets the policy change without touching loan code.
- **Observer (overdue notifications, reservation availability)** — when a book is returned, the next person in the reservation queue gets notified. When a loan goes overdue, the member is emailed. Both fit Observer cleanly.

## Class diagram (sketch)

```text
                  ┌──────────┐
                  │ Library  │ «Singleton»
                  └─────┬────┘
              ┌─────────┼──────────┬─────────┐
              ◆         ◆          ◆         ◆
              │         │          │         │
          ┌───▼──┐  ┌───▼────┐ ┌───▼────┐ ┌──▼─────┐
          │Catalog│ │Members │ │Loans   │ │Reserv. │
          └───┬──┘  └────────┘ └────────┘ └────────┘
              │
              │  (search index)
              ▼
          ┌────────┐  1     *  ┌──────────┐
          │  Book  │◆─────────│ BookItem │
          └────────┘           └──────────┘
                                    │
                                    │ involved in
                                    ▼
                             ┌─────────────┐
                             │    Loan     │
                             ├─────────────┤
                             │ -member     │
                             │ -dueDate    │
                             └──────┬──────┘
                                    │
                                    ▼
                              FineStrategy «interface»
```

## Code sketch

```java
class Book {                         // logical book — one per ISBN
    private final String isbn;
    private final String title;
    private final String author;
    // immutable — books don't get retitled
}

class BookItem {                     // physical copy
    private final String barcode;
    private final Book book;
    private BookStatus status = BookStatus.AVAILABLE;

    public synchronized boolean checkout(Member m, LocalDate due) {
        if (status != BookStatus.AVAILABLE) return false;
        this.status = BookStatus.LOANED;
        Library.get().recordLoan(new Loan(this, m, due));
        return true;
    }

    public synchronized void returnItem() {
        this.status = BookStatus.AVAILABLE;
        Library.get().onReturn(this);   // fires observers (reservation queue)
    }
}

interface FineStrategy {
    double computeFine(Loan loan, LocalDate now);
}

class FlatPerDayFine implements FineStrategy {
    public double computeFine(Loan loan, LocalDate now) {
        long daysLate = Math.max(0, ChronoUnit.DAYS.between(loan.getDueDate(), now));
        return daysLate * 1.0;
    }
}

class Library {
    private static final Library INSTANCE = new Library();
    private final Catalog catalog = new Catalog();
    private final Map<String, Member> members = new HashMap<>();
    private final List<Loan> loans = new ArrayList<>();
    private FineStrategy fineStrategy = new FlatPerDayFine();

    public static Library get() { return INSTANCE; }

    public List<Book> search(String query) { return catalog.search(query); }

    public Loan issueBook(Member m, BookItem item) {
        if (!item.checkout(m, LocalDate.now().plusDays(14))) {
            throw new ItemUnavailableException();
        }
        return loans.get(loans.size() - 1);
    }
}
```

## Key design decisions

1. **`Book` vs `BookItem` separation** — the single most important call. The logical work (`Book`) and the physical copy (`BookItem`) have different lifecycles, different identifiers, and different responsibilities. Conflating them makes everything else worse.
2. **Singleton `Library`** — one logical library; gives librarians and members a stable entry point. Internally composed of subsystems (`Catalog`, member registry, loan registry) for SRP.
3. **`Catalog` abstraction** — wraps the search index. Today it might be an in-memory hash; tomorrow it's Elasticsearch. The interface stays the same (DIP).
4. **`FineStrategy` as a separate object** — fine policy is *governance*, not core lending logic. Externalizing it lets policy change without touching `Loan` or `Library`.
5. **Synchronization on `BookItem`** — a copy can be checked out by exactly one member at a time. Locking at the item level avoids a global library lock.
6. **Observer on returns** — notifying the next reservation holder is asynchronous and out-of-band; subscriber pattern keeps `BookItem.returnItem()` short.

## Extensibility

**Add e-books / audiobooks:**

1. Introduce `DigitalBookItem` that doesn't hold physical state; checkouts grant time-limited download URLs.
2. The `BookItem` interface (or abstract base) absorbs the variance; `Member.borrow()` doesn't change.

**Add a new fine policy (escalating):**

1. Implement `EscalatingFine implements FineStrategy` (e.g., $1/day for first week, $2/day after).
2. Inject it into the library. No edits to existing code.

**Add reservations:**

1. Add a `ReservationQueue` per `Book` (the *logical* book, not item).
2. On `BookItem.returnItem()`, the library's observer pops the queue and notifies the next member.
3. The borrow flow becomes "reserved member has priority for N hours, then the next person".

**Switch search to Elasticsearch:**

1. Implement `ElasticCatalog implements Catalog`.
2. Swap the wiring. Search clients don't change.

## Linked concepts

- [OOP](../fundamentals/oop.md) — separating logical vs physical models is an OOP modeling skill.
- [SOLID](../fundamentals/solid.md) — SRP across `Library`, `Catalog`, `FineStrategy`; OCP via the strategy and abstract item.
- [Design Patterns](../fundamentals/design-patterns.md) — Singleton, Factory, Strategy, Observer.
